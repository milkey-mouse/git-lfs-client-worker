# Git LFS Client Worker

This [middleware](https://developers.cloudflare.com/pages/platform/functions/middleware/) allows [Cloudflare Pages](https://pages.cloudflare.com/) sites built from a Git repository to serve large files directly from the repository's [Git LFS](https://git-lfs.com/) server, transparently overcoming the [25 MiB](https://developers.cloudflare.com/pages/platform/limits/#file-size) Pages file size limit.

# Usage

We'll assume you already have a Git repository to publish on Cloudflare Pages, and your working directory is the root of your repository:

    cd "$(git rev-parse --show-toplevel)"


#### Install Git LFS

If you haven't used Git LFS before, you may need to install it. Run the following command:

    git lfs version

If your output includes `git: 'lfs' is not a git command`, then follow the Git LFS [installation instructions](https://github.com/git-lfs/git-lfs#installing).

#### Install smudge and clean filters

Even if the Git LFS binary was already installed, the smudge and clean filters Git LFS relies upon may not be. Ensure they are installed for your user account:

    git lfs install


#### Install the LFS Client Worker

The LFS Client Worker works like the [standard Git LFS client](https://git-lfs.com/): when your Pages site is about to serve an LFS "pointer" file, it looks up the LFS object it points to and serves it instead. To do this, the LFS Client Worker needs to be run as "[middleware](https://developers.cloudflare.com/pages/platform/functions/middleware/)" on your site, which means `_middleware.js` from this repo should end up as `functions/_middleware.js` in your repo. It also needs a symlink from `functions/.lfsconfig.txt` to `.lfsconfig` to load `.lfsconfig` correctly. If you don't have other functions on your Pages site (i.e. there is no `functions` directory in the root of your project), the simplest way to install LFS Client Worker this is to add this repo to your site as a submodule called `functions`:

    git submodule add https://github.com/milkey-mouse/git-lfs-client-worker functions
    git commit -m "Add Git LFS Client Worker"


#### Disable smudge and clean filters by default

The worker will replace LFS "pointers" with the underlying objects on the fly, so we don't want the standard Git LFS client to do so when the repo is cloned by Cloudflare Pages. (Otherwise, Cloudflare Pages builds will fail even when LFS stores all large files.) Environment variables don't seem to influence the "Cloning git repository" step in Pages builds, so smudge and clean filters must be disabled by default in `.lfsconfig`:

    git config -f .lfsconfig lfs.fetchexclude '*'
    git add .lfsconfig
    git commit -m "Disable LFS fetching by default"

When a commit including this `.lfsconfig` change is checked out, Git LFS will not replace LFS pointers with the objects they point to. On your own copies of the repo, override `lfs.fetchexclude` to continue doing so:

    git config lfs.fetchexclude ''

#### Change LFS servers

GitHub and GitLab both provide default LFS servers, but they are not ideal for this use case:

- Storage space is limited.
  - You can store up to 1 GiB of LFS objects across all repositories on GitHub's free tier. Additional storage costs $0.10/GiB-month [bundled with bandwidth](https://docs.github.com/en/billing/managing-billing-for-git-large-file-storage/about-billing-for-git-large-file-storage#purchasing-additional-storage-and-bandwidth).
  - You can store up to 5 GB on GitLab's free tier, but this includes [all other GitLab services](https://about.gitlab.com/pricing/faq-paid-storage-transfer/#q-what-constitutes-storage-usage). Additional storage must be bought in $60/year "[packs](https://about.gitlab.com/pricing/faq-paid-storage-transfer/#purchasing-additional-storage-and-transfer)".
- Bandwidth is limited.
  - You can download up to 1 GiB/month of LFS objects across all repositories on GitHub's free tier. Additional bandwidth costs $0.10/GiB [bundled with storage](https://docs.github.com/en/billing/managing-billing-for-git-large-file-storage/about-billing-for-git-large-file-storage#purchasing-additional-storage-and-bandwidth).
  - You can download up to 10 GB/month on GitLab's free tier, but this includes [all other GitLab services](https://about.gitlab.com/pricing/faq-paid-storage-transfer/#q-what-constitutes-transfer-usage). Additional bandwidth must be bought in $60/year "[packs](https://about.gitlab.com/pricing/faq-paid-storage-transfer/#purchasing-additional-storage-and-transfer)".
- Latency is questionable.
  - GitHub's LFS server takes ~270ms to reply with an object's URL. The actual object store takes another 20-100ms to reply.
  - Neither GitHub nor GitLab built their LFS servers with serving web content in mind (which, to be fair, is slightly cursed).

**Consider [setting up LFS S3 Proxy](https://github.com/milkey-mouse/git-lfs-s3-proxy)** with [R2](https://developers.cloudflare.com/r2) as your LFS server instead. On Cloudflare's free tier, it can serve up to 10 GB of LFS objects with unlimited bandwidth and the lowest possible latency (your objects are in the same datacenters as the LFS Client Worker). If you have more than 10 GB of assets, additional storage is $0.015/GB-month, several times cheaper than GitHub or GitLab.

If you decide to use the default LFS server, you'll still want to [explicitly specify its URL in `.lfsconfig`](https://github.com/milkey-mouse/git-lfs-s3-proxy#private-repo) as the LFS Client Worker has no other way of knowing it (workers don't know what repo their Pages site was built from).

#### Add files to Git LFS

You're now ready to [start using Git LFS](https://github.com/git-lfs/git-lfs#example-usage). At the very least, you should add all files larger than the 25 MiB Cloudflare Pages limit to Git LFS:

    find . -type f '!' -path './.*' -size +25M -exec git lfs track {} +
    find . -type f '!' -path './.*' -size +25M -exec git add --renormalize {} +
    git add .gitattributes
    git commit -m "Add files over 25 MiB to Git LFS"

If you add more large files in the future, add them to Git LFS before you commit them with `git lfs track`:

    git lfs track bigfile

If certain file types are consistently larger than 25 MiB, you can automatically track them with Git LFS:

    git lfs track '*.mp4'


#### Create a Pages site

If you haven't already, [create a Cloudflare Pages site](https://developers.cloudflare.com/pages/get-started/guide/) from your repo. If you have, push your changes to trigger a rebuild of your site:

    git push


#### Optional: Bind your R2 bucket to LFS Client Worker

Once your Pages site is set up, if you changed your LFS server to [LFS S3 Proxy](https://github.com/milkey-mouse/git-lfs-s3-proxy) with [R2](https://developers.cloudflare.com/r2), you should [add a binding to your Pages site](https://developers.cloudflare.com/pages/platform/functions/bindings/#r2-buckets) for slightly better performance:

- Open the [Workers & Pages](https://dash.cloudflare.com/?to=/:account/pages) section of the Cloudflare dashboard.
- Select your Pages site.
- Set up `LFS_BUCKET`:
  - Navigate to **Settings** > **Functions** > **R2 bucket bindings** > **Production**.
  - Click **Add binding**.
  - Set **Variable name** to `LFS_BUCKET`.
  - For **R2 bucket**, select the bucket you created for LFS S3 Proxy.
  - Click **Save**.
- Set up `LFS_BUCKET_URL`:
  - Navigate to **Settings** > **Environment variables** > **Production**.
  - click **Add variables**.
  - Set **Variable name** to `LFS_BUCKET_URL`.
  - Set **Value** to your [LFS server URL](https://github.com/milkey-mouse/git-lfs-s3-proxy#find-your-lfs-server-url) *without the access key* (just `https://<INSTANCE>/<ENDPOINT>/<BUCKET>`).
- Re-deploy your Pages site:
  - Navigate to **Deployments** > **Production** > **View details**.
  - Click **Manage deployment** > **Retry deployment**.

With these variables set, LFS Client Worker can skip asking LFS S3 Proxy for [presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/) and fetch objects directly from the bucket.
