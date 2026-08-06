> [!NOTE]
> This repository was branched out of [storiny/web](https://github.com/storiny/web) which was originally a monolith. If you're looking for the original git history before the split, you can find it over on the original repository.

# Storiny CDN

Content delivery network for media assets on Storiny. Built on top of `nginx` with `njs` (to perform application-specific logic & routing), it acts as a reverse proxy sitting directly in front of our AWS S3 storage buckets and [imgproxy](https://github.com/imgproxy/imgproxy) instance.

---

## Tech stack

- **Web server**: [nginx](https://nginx.org/) handles all incoming HTTP requests, caching, and SSL termination
- **Edge logic**: [njs](https://nginx.org/en/docs/njs/) powers our dynamic request routing, URL signature verification, and path rewriting
- **Image processing**: Requests are securely rewritten and forwarded to an internal [imgproxy](https://imgproxy.net/) container for resizing and format conversion
- **Storage**: AWS S3 serves as the primary object storage for internal media assets and user uploads
- **Tooling**: [Rollup](https://rollupjs.org/) and Babel are used to bundle our TypeScript `njs` handlers into the strict ES5 subset required by the `nginx`

---

## Under the hood

1. **Request interception**: `nginx` captures incoming requests for media assets and passes them to our bundled `handler.js`
2. **Dynamic routing**: The `njs` script parses the URI to extract the requested width, asset type, and object key using a regular expression
3. **URL signing &  verification**: For remote images, it decodes a hex-encoded URL and validates its cryptographic signature (digest) to prevent our imgproxy instance from being abused as an open proxy
4. **imgproxy rewriting**: It translates client-facing URLs (like `/uploads/600/image.jpg`) into imgproxy processing paths (like `@proxy_pass /internal/resize:fit:600:0:0/extend_ar:false:ce:0:0/plain/s3://uploads-bucket/image.jpg`)
5. **Smart defaults**: Specific routes apply smart defaults automatically. For example, `dl` routes inject `return_attachment:true`, `mail-assets` force a conversion to `png` for email client compatibility and `.ico` requests are forcefully resized to `48x48`

---

## API endpoints

- `GET /health`: Health check endpoint. Proxied directly to the upstream `imgproxy` health check
- `GET /remote/{width}/{digest}/{hex}`: Fetches and optimizes a third-party remote image securely by validating its signature and decoding the hex URL
- `GET /thumb/{key}`: A specialized thumbnail route that forcefully resizes an S3 upload to `720x404` for displaying as a thumbnail
- `GET /uploads/{width}/{key}`: Dynamically resizes a user-uploaded image from the S3 bucket to the specified width
- `GET /dl/{width}/{key}`: Same as uploads, but forces the browser to download the file by appending attachment headers
- `GET /web-assets/raw/{key}`: Serves raw base S3 assets directly without `imgproxy` manipulation
- `GET /mail-assets/{width}/{key}`: Fetches a base asset and forcefully converts it to PNG for safe rendering in email clients

---

## Running locally

```bash
# clone the repository
git clone https://github.com/storiny/cdn.git
cd cdn

# install deps
yarn install

# bundle the njs scripts, spin up nginx & imgproxy via docker
yarn dev
```
