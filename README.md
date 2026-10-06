# Blog Application

A server-rendered blog application built with Node.js, Express 5, EJS, and MongoDB/Mongoose. Users can register and sign in, create blog posts with cover-image uploads, view posts, and comment while signed in.

The application is suitable for deployment to AWS Elastic Beanstalk as a Node.js web application. It uses `app.js` as its entry point and `npm start` as its production start command.

## Project and deployment analysis

| Item | Current implementation |
| --- | --- |
| Application | Node.js / CommonJS, Express 5 |
| Rendering | EJS templates under `views/`; this is a web application, not a JSON-only API |
| Entry point | `app.js` (`package.json` also has `"main": "app.js"`) |
| Production command | `npm start` runs `node app.js` |
| Development command | `npm run dev` runs `nodemon app.js` |
| Database | Mongoose connects using `process.env.MONGO_URL` |
| Port | `app.js` reads `process.env.PORT`, falling back to `8000` for local development |
| Authentication | JSON Web Tokens in a `token` cookie; signing key is `JWT_SECRET` |
| Uploads | Multer writes files into `public/uploads/`, served as static files |

Only `PORT`, `MONGO_URL`, and `JWT_SECRET` are read as application environment variables. `dotenv` loads a local `.env` file at the start of `app.js`. On AWS, Elastic Beanstalk injects the configured environment variables into `process.env`; the deployed application does not need to read a deployed `.env` file.

The code starts the HTTP listener only after a successful MongoDB connection and fails startup with an explicit message if `MONGO_URL` or `JWT_SECRET` is missing. No `Procfile`, `.ebextensions` file, or custom port mapping is needed for the default Elastic Beanstalk Node.js platform: it detects the package `start` script, configures its reverse proxy, and provides the `PORT` value to the Node.js process.

### Routes to test

- `GET /` — home page and blog list.
- `GET /user/signup`, `POST /user/signup` — registration.
- `GET /user/signin`, `POST /user/signin` — sign in.
- `GET /user/logout` — sign out.
- `GET /blog/add-new` — signed-in user's post form.
- `POST /blog` — create a post and upload its cover image.
- `GET /blog/:id` — post detail and comments.
- `POST /blog/comment/:blogId` — create a comment.

There is no separate JSON API route in the current source.

## Prerequisites

For local development:

- Node.js **20.19 or newer**. The installed Mongoose dependency declares a minimum of Node.js 20.19.
- npm (included with Node.js).
- A local MongoDB server, or a MongoDB Atlas database.
- A terminal opened in the project directory.

For AWS:

- An AWS account with permission to use Elastic Beanstalk, EC2, IAM, VPC, and (if used) Secrets Manager.
- AWS CLI v2, installed and configured for the intended account and region.
- Elastic Beanstalk CLI (`eb`). Install it with Python/pip, for example `py -m pip install --user awsebcli` on Windows; ensure the installed `eb` command is on `PATH`.
- The Node.js and npm prerequisites above.
- MongoDB Atlas project permissions to create a cluster, database user, and network access entry.

Elastic Beanstalk, EC2, NAT gateways, load balancers, and Atlas clusters may incur charges. Review current AWS and MongoDB Atlas pricing before creating resources. A single-instance EB environment avoids a load balancer but is not highly available.

## Local development

1. Install project dependencies:

   ```powershell
   npm ci
   ```

2. Copy `.env.example` to `.env`:

   ```powershell
   Copy-Item .env.example .env
   ```

3. Ensure MongoDB is running locally. The example URL selects the local `BlogApp` database:

   ```dotenv
   MONGO_URL=mongodb://127.0.0.1:27017/BlogApp
   PORT=8000
   JWT_SECRET=<your-random-local-secret>
   ```

   Generate a random local key in PowerShell and put the output in `.env`:

   ```powershell
   node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
   ```

   Keep the angle-bracket placeholder out of the actual `.env`; use the generated value instead.

4. Start in development or production mode:

   ```powershell
   npm run dev
   # or
   npm start
   ```

5. Open `http://localhost:8000/`. Register, sign in, create a post, upload a cover image, and add a comment to exercise the main flows.

`PORT` is optional locally because the application falls back to `8000`. `MONGO_URL` and `JWT_SECRET` are required; the application exits with an error if either is missing.

## Environment variables

| Name | Required? | Local development | Elastic Beanstalk |
| --- | --- | --- | --- |
| `MONGO_URL` | Yes | `mongodb://127.0.0.1:27017/BlogApp`, if running MongoDB locally | MongoDB Atlas `mongodb+srv://...` connection string pointing at the production database |
| `JWT_SECRET` | Yes | A random local-only secret | A different, strong production secret; preferably retrieved from AWS Secrets Manager |
| `PORT` | No when running locally; supplied by EB | Set to `8000` or leave unset to use the fallback | Elastic Beanstalk's Node.js platform sets this for the app/proxy. Do not hardcode it or normally set it manually |

Local and production values must not be confused:

```dotenv
# Local .env (not committed)
PORT=8000
MONGO_URL=mongodb://127.0.0.1:27017/BlogApp
JWT_SECRET=<different-random-local-secret>
```

```text
# Elastic Beanstalk application environment
PORT=<set by the Elastic Beanstalk Node.js platform>
MONGO_URL=<MongoDB Atlas production connection string>
JWT_SECRET=<separate production secret>
```

`PORT` should continue to be read as `process.env.PORT` (the application already does this) with a local fallback. Do not set the production listener to a fixed value such as `8000`: the EB proxy routes to the port supplied in `process.env.PORT`. The EB Node.js platform supplies this variable; it does not need to be manually configured in the EB environment.

`MONGO_URL` is the existing application variable name. Reuse that name and change only its **value** in production to the Atlas URI; no MongoDB-specific code change is needed. Never use the local `127.0.0.1` URI on EB because it would refer to the EC2 instance itself, not your computer or Atlas.

### Configure values in the Elastic Beanstalk console

1. Open **Elastic Beanstalk → Environments → your environment → Configuration**.
2. Under **Updates, monitoring, and logging** (or the software configuration section shown for the selected platform), choose **Edit**.
3. In **Environment properties**, add `MONGO_URL` and `JWT_SECRET`. Do not add a hand-picked `PORT`.
4. Save/apply the configuration and wait for the environment update to finish.

For a quick non-production trial, environment properties can be entered directly. Treat these as sensitive: principals with appropriate EB permissions can inspect environment settings, and avoid pasting secrets into shared terminals or shell-history. For production, store each value in AWS Secrets Manager and configure EB to expose the secret by ARN as an environment variable (choose the Secrets Manager value type in the console). The EB EC2 instance profile must allow `secretsmanager:GetSecretValue` on those secrets and, for a customer-managed KMS key, the required `kms:Decrypt`. Use an Elastic Beanstalk platform version that supports environment secrets. Never put secret contents in source code, `.env.example`, Git, or a committed EB config file.

### Configure with AWS CLI

For ordinary environment properties, the EB CLI offers `eb setenv`, and the AWS CLI offers `update-environment`. These commands take the secret value as an argument and may expose it in command history/process inspection. Prefer the Secrets Manager reference option for production rather than placing credentials directly on the command line.

To set **secret references** from Secrets Manager using AWS CLI (PowerShell line continuation shown), replace the non-secret placeholders with the environment name, region, account, and secret ARNs:

```powershell
aws elasticbeanstalk update-environment `
  --environment-name <EB_ENVIRONMENT_NAME> `
  --option-settings `
  "Namespace=aws:elasticbeanstalk:application:environmentsecrets,OptionName=MONGO_URL,Value=<MONGO_URL_SECRET_ARN>" `
  "Namespace=aws:elasticbeanstalk:application:environmentsecrets,OptionName=JWT_SECRET,Value=<JWT_SECRET_SECRET_ARN>" `
  --region <AWS_REGION>
```

Use the same namespace in the EB console's Secrets Manager-backed environment variable settings. EB retrieves the values when bootstrapping instances; ensure the instance profile has the permissions above. This configuration does not change the names read by the Node.js code.

## MongoDB Atlas production setup

1. Sign in to MongoDB Atlas and create a project for this application.
2. Create a production cluster in a region geographically close to the Elastic Beanstalk region. Choose a tier appropriate for expected traffic and review its cost.
3. Under **Database Access**, create a dedicated database user for this application. Use a generated strong password and the least privileges appropriate to this blog app (typically read/write access to the application's database, not an Atlas project administrator). Store the credentials only in Secrets Manager.
4. Under **Network Access**, allow inbound connections from the application. Do not assume that an EB instance's public IP remains fixed:
   - For production allow-listing of a public Atlas endpoint, route EB instance egress through a VPC NAT gateway with an Elastic IP and allow-list that static public IP in Atlas. NAT gateways have an ongoing charge.
   - Atlas/AWS private connectivity (such as VPC peering or a private endpoint where available) is another production option and avoids public database ingress; configure the matching Atlas and AWS network routes/security rules.
   - For a short-lived test only, Atlas allows `0.0.0.0/0`; it exposes the database endpoint to connection attempts from anywhere and should not be left in place for production. Strong credentials do not make this a recommended production network policy.
5. In Atlas, open the cluster's **Connect** flow, choose **Drivers**, and copy the SRV URI. Replace its username/password placeholders with the database user's credentials (URL-encode reserved characters in the password), and select a database name, for example `BlogApp`:

   ```text
   mongodb+srv://<DATABASE_USER>:<URL_ENCODED_PASSWORD>@<CLUSTER_HOST>/BlogApp?retryWrites=true&w=majority
   ```

6. Save the completed URI as the value of the production `MONGO_URL` secret. Never save it into source code or the committed README. Confirm Atlas network access and credentials before deploying.

The local URL in `.env` is `mongodb://127.0.0.1:27017/BlogApp`; the production value is the Atlas `mongodb+srv://...` URI. Mongoose already connects with `mongoose.connect(process.env.MONGO_URL)`, so the variable is reusable as-is.

## AWS Elastic Beanstalk deployment

### Step 1: Prepare the project

1. Confirm the project contains `app.js`, `package.json`, and `package-lock.json` at the project root. They are present in this project.
2. Keep `npm start` as the start command; it already runs `node app.js`. Do not add a `Procfile` unless you intentionally change the start procedure.
3. Keep `.env` local. `.gitignore` now excludes `.env` and `.env.*`, while allowing the safe `.env.example` template.
4. Generate and store the production `JWT_SECRET` separately from your local key.
5. Before the first deployment, make sure the production Atlas cluster, user, and network access are ready, then install/verify the AWS CLI and EB CLI.

### Step 2: Configure MongoDB Atlas

Complete the Atlas steps above and put the final connection URI in Secrets Manager as a secret, for example a secret named for the production `MONGO_URL`. Keep its ARN; you will configure EB to expose it as `MONGO_URL`.

### Step 3: Configure AWS and Elastic Beanstalk

1. Configure AWS CLI credentials using an IAM identity authorized for the required deployment operations. Prefer IAM Identity Center or short-lived credentials over long-lived access keys. Select the same AWS region for the application and environment:

   ```powershell
   aws configure
   aws sts get-caller-identity
   ```

2. From the project root, initialize EB CLI. `package.json` does not pin a Node.js version; the installed Mongoose dependency requires Node.js 20.19 or newer. Choose the intended region and a currently supported **Node.js LTS on 64-bit Amazon Linux 2023** platform branch that satisfies that minimum. Platform availability changes, so choose a supported branch in the prompt rather than copying an obsolete platform version.

   ```powershell
   eb init blog-application
   ```

3. Create the web server environment. A single-instance environment is simpler/cheaper for a first deployment but has no load balancer or high availability:

   ```powershell
   eb create blog-application-prod --single
   ```

   For production availability, omit `--single` and use a load-balanced environment with suitable scaling, health checks, and networking. Ensure the EC2 instance profile is present. If using Secrets Manager, attach permissions to that **instance profile** (not merely your local deployer's IAM identity).

### Step 4: Configure production environment variables and networking

1. In EB **Configuration → Software / Environment properties**, define `MONGO_URL` and `JWT_SECRET` as Secrets Manager references, or use the `environmentsecrets` namespace shown above. Make sure the deployed platform version supports this feature. Do not manually configure `PORT`.
2. For a custom Atlas network allow-list in production, configure the EB VPC/subnets and stable egress through NAT with an Elastic IP, or configure private connectivity. Add only the static NAT IP or required private network range to Atlas.

### Step 5: Deploy the application

From the directory containing `app.js` and `package.json`:

```powershell
eb deploy blog-application-prod
```

The EB Node.js platform detects the `start` script and runs `npm start`. Keep `package-lock.json` so installs are reproducible. Do not upload local `node_modules`; EB installs dependencies from the package manifests. The ignore rules help prevent `.env` and dependencies from entering the source bundle.

To deploy later changes, run `eb deploy` again from the project root. Do not deploy a local `.env` file or put Atlas credentials in an EB source bundle.

### Step 6: Verify the deployment

1. Check health and events:

   ```powershell
   eb status blog-application-prod
   eb health blog-application-prod
   eb events blog-application-prod
   ```

   Or with AWS CLI:

   ```powershell
   aws elasticbeanstalk describe-environments --environment-names blog-application-prod --region <AWS_REGION>
   ```

2. Obtain the environment URL:

   ```powershell
   eb open blog-application-prod
   ```

   Or inspect the environment's `CNAME` with `describe-environments`.

3. Open the URL in a browser and confirm the home page renders.
4. Check the home page from PowerShell:

   ```powershell
   Invoke-WebRequest https://<EB_ENVIRONMENT_CNAME>/
   ```

   Replace the host with the HTTP/HTTPS URL shown by EB. A successful response should have an HTTP 200 status. The current application serves HTML pages; there is no JSON health/API endpoint.

### Step 7: Test the application

In a browser, exercise the actual server-rendered application flows:

1. Open `/user/signup` and create a test user.
2. Sign in at `/user/signin`.
3. Open `/blog/add-new`, create a post, and upload a cover image.
4. Open the new post and add a comment while signed in.
5. Sign out, then verify the home page and post pages still render when logged out.

If these requests fail, view EB logs and environment events:

   ```powershell
   eb logs blog-application-prod
   ```

For a bundle of recent logs:

   ```powershell
   eb logs blog-application-prod --all
   ```

### Production limitation: uploaded images

Multer currently writes cover images to `public/uploads/` on the EB instance's local filesystem. That works for a basic single-instance demonstration, but local instance storage is not durable/shared: images may disappear when an instance is replaced or the application is redeployed, and multiple instances do not share uploaded files. The checked-in image files in that directory are part of the local project, not durable production upload storage. For a production multi-instance deployment, migrate uploads to Amazon S3 (or another shared object store) and store object URLs in MongoDB before relying on uploads.

### Step 8: Troubleshoot common issues

| Symptom | Check / resolution |
| --- | --- |
| Environment fails to start: `MONGO_URL environment variable is required` | Set the EB `MONGO_URL` environment variable or secret reference. Do not use a local `.env` on the instance. |
| Environment fails to start: `JWT_SECRET environment variable is required` | Add a strong `JWT_SECRET` secret/reference in EB. Do not restore a hardcoded key. |
| `MongoServerSelectionError`, timeouts, or Atlas connection refusal | Verify Atlas cluster is running, URI hostname/user/password/database are correct, the password is URL-encoded, Atlas Network Access allows EB's actual stable egress IP/CIDR, and VPC routes/security groups permit outbound database traffic. |
| Atlas authentication fails | Confirm you created a **database user** (not only an Atlas website user), the password is correct, and the URI points at the intended database. Rotate a password if it was exposed. |
| EB reports `502 Bad Gateway` or health is red | Read `eb events` and `eb logs`. Confirm `npm start` works, `app.js` is at source-bundle root, `PORT` is read from `process.env.PORT`, and the startup MongoDB connection succeeds. |
| EB reports no application / start command failure | Confirm root-level `package.json` and `package-lock.json` are included and `npm start` is defined. It is already defined as `node app.js`. Check npm install errors in EB logs. |
| Works locally but not on EB | Local `.env` is not automatically deployed. Configure `MONGO_URL` and `JWT_SECRET` in EB; check Node.js platform compatibility and Atlas network access. |
| Image upload fails or images later disappear | Confirm `public/uploads/` exists and can be written. Local instance storage is temporary/non-shared; use S3 for production durability and multi-instance use. |
| Browser renders sign-in as logged out after changing JWT key | Existing cookies were signed using the prior key. Users must sign in again after the key is rotated. |
| `npm`/`eb` command is not recognized in PowerShell | Reopen PowerShell after installation and add Node/npm or the Python user Scripts directory to `PATH`. On Windows, `npm.cmd` can be used if PowerShell blocks `npm.ps1`. |
| Secret reference does not resolve | Verify the EB platform supports environment secrets, ARN and region are correct, the instance profile has `secretsmanager:GetSecretValue` and any required `kms:Decrypt`, then inspect EB events/logs. |

## Security checklist

- `.env` is ignored by the new `.gitignore`; `.env.example` contains only local defaults and an empty JWT key.
- The old JWT signing key was hardcoded in `services/authentication.js`. It has been removed from source and must be configured as `JWT_SECRET`. Treat the former value as exposed: rotate it anywhere reused and invalidate old login cookies/tokens by using a new production key.
- No MongoDB Atlas credentials are included in the README or example environment file. If real credentials were ever committed or shared, rotate them; deleting the text alone does not revoke the credential.
- Keep production credentials in AWS Secrets Manager and restrict secret access by IAM. Do not place secret values in Git, command history, screenshots, support tickets, or deployment bundles.
- Use TLS for web traffic, restrictive Atlas network access, least-privilege database users and IAM roles, and regularly rotate credentials.
- Configure secure cookie options and HTTPS behavior before exposing authentication to production users; the current `res.cookie("token", token)` does not explicitly set `secure`, `httpOnly`, or `sameSite`.
- The current password hashing uses a fast SHA-256 HMAC rather than a password-specific KDF, and JWTs are not assigned an expiry. Review these authentication choices and migrate to a password KDF (such as Argon2id/bcrypt) and expiring tokens before treating the app as a hardened production service.
- Move uploads to S3 before using multiple instances or depending on persistence.

## Useful references

- [Elastic Beanstalk Node.js platform](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create_deploy_nodejs.container.html)
- [Elastic Beanstalk Node.js Express deployment](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/create_deploy_nodejs_express.html)
- [Elastic Beanstalk environment properties](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/environments-cfg-softwaresettings.html)
- [Elastic Beanstalk secrets and parameters](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/AWSHowTo.secrets.html)
- [MongoDB Atlas IP access list](https://www.mongodb.com/docs/atlas/security/ip-access-list/)
- [MongoDB Atlas database users](https://www.mongodb.com/docs/atlas/security-add-mongodb-users/)
# Blog-Application
