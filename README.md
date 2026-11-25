# AWS CloudFront + S3 (OAC) Practice Lab

Host a static page on a **private S3 bucket** and serve it through **CloudFront (CDN)**.

## Cost

- CloudFront has an always-free tier (1 TB data out, 10M requests per month). Check the AWS pricing page for current limits.
- A tiny S3 file costs almost nothing.
- Keep **WAF off**. It costs money.

## Architecture

```
User → CloudFront (edge location) → S3 bucket (private, via OAC)
```

## Prerequisites

- AWS account
- AWS CLI (optional, for testing and invalidation)
- A simple `index.html` file

---

## Step 1: Create the S3 bucket

1. Open **S3** → **Create bucket**.
2. Bucket name: `<your-bucket-name>` (no dots in the name).
3. Region: `ap-south-2` (or your choice).
4. Keep **Block all public access = ON**.
5. Create the bucket.

## Step 2: Upload `index.html`

1. Open the bucket → **Upload**.
2. Add `index.html` at the **root** (not inside a folder).
3. Upload.

## Step 3: Create the CloudFront distribution

1. Open **CloudFront** → **Create distribution**.
2. **Origin domain:** use the S3 **REST endpoint**:

   ```
   <your-bucket-name>.s3.<region>.amazonaws.com
   ```

   Or pick the bucket from the dropdown.

   | Correct | Wrong |
   |---|---|
   | `my-bucket.s3.ap-south-2.amazonaws.com` | `s3://my-bucket` |
   | | `my-bucket` |
   | | `my-bucket.s3-website.ap-south-2.amazonaws.com` |

3. **Origin access:** choose **Origin access control settings (recommended)**.
4. Click **Create new OAC** (or select the existing one).
5. **Default root object:** `index.html`
6. **WAF:** turn **off**.
7. **Price class** (if shown): choose the lowest-cost option for practice.
8. Create the distribution.

## Step 4: Add the bucket policy

CloudFront needs permission to read from S3.

**Copy the policy**

- In the distribution, click **Copy policy** (blue banner or Origins → Edit).

**Paste the policy**

1. Open S3 → your bucket → **Permissions** tab.
2. Scroll to **Bucket policy** → **Edit**.
3. Paste the policy → **Save changes**.

Example policy:

```json
{
  "Version": "2008-10-17",
  "Id": "PolicyForCloudFrontPrivateContent",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::<your-bucket-name>/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::<account-id>:distribution/<distribution-id>"
        }
      }
    }
  ]
}
```

**If a policy already exists**

- Do not paste a second full policy.
- Add only the new `{ ... }` statement inside the existing `Statement` list.
- Separate statements with a comma.
- Do not keep two statements with the same `Sid`.

## Step 5: Keep the bucket private

- S3 → bucket → Permissions → **Block public access** must be **On**.
- This does not break CloudFront. The policy allows only the CloudFront service.
- Remove any policy statement with `"Principal": "*"`.

## Step 6: Test the site

1. Wait until the distribution status is **Deployed** (about 5 minutes).
2. Open in the browser:

   ```
   https://<distribution-id>.cloudfront.net
   ```

## Step 7: Test caching

```bash
curl -I https://<distribution-id>.cloudfront.net
```

Run it twice.

| Request | Header |
|---|---|
| First | `X-Cache: Miss from cloudfront` |
| Second | `X-Cache: Hit from cloudfront` |

## Step 8: Check the edge location

You cannot choose the edge location. CloudFront picks the nearest one.

Check which edge served you:

```bash
curl -I https://<distribution-id>.cloudfront.net
```

Look at `X-Amz-Cf-Pop`. Example: `MAA50-C1` (MAA = Chennai).

## Step 9: Practice cache invalidation

1. Edit `index.html` and upload it again.
2. Refresh the site. The old version may still show (cached).
3. Run an invalidation:

```bash
aws cloudfront create-invalidation \
  --distribution-id <distribution-id> \
  --paths "/*"
```

4. Refresh again. The new version shows.

## More practice ideas

- Change the TTL in a cache policy.
- Add a custom domain with an ACM certificate (must be in **us-east-1**).
- Add a custom error page (403/404).

---

## Troubleshooting

| Error | Fix |
|---|---|
| `The parameter Origin DomainName does not refer to a valid S3 bucket` | Use the REST endpoint `bucket.s3.region.amazonaws.com`. Do not use `s3://`. |
| `AccessDenied` (XML) right after setup | Wait 2–5 minutes. CloudFront and the policy need time to apply. |
| `AccessDenied` that stays | Check the bucket policy is saved, the distribution ID matches, `index.html` is at the bucket root, and OAC is selected. |
| `AccessDenied` on the base URL only | Set **Default root object** to `index.html`. |
| JSON error when saving the policy | You pasted a full second policy. Add only the statement block, with a comma. |
| Old page still shows after upload | Run an invalidation (Step 9). |

---

## Cleanup (avoid charges)

1. CloudFront → select the distribution → **Disable**.
2. Wait until it is disabled → **Delete**.
3. S3 → empty the bucket → delete the bucket.
4. Set an **AWS Budgets** alert at $1.

## Warnings

- Do not enable WAF, Lambda@Edge, or Shield Advanced. They cost extra.
- Never make the bucket public. Use OAC.
