# This doc tracks the installation of a PoC of BACKSTAGE.


## SOURCES
    - https://backstage.io/docs/getting-started/
    - https://www.kosli.com/blog/implementing-backstage-1-getting-started/
    - install build-essentials under alma9 --> https://linux.how2shout.com/how-to-install-build-tools-on-almalinux-or-rockylinux-8-9/



## Tasks:

### Install prerequisites:


__SYSTEM__

    - Install ALMA 9.6 minimal
    - set the hostname & IP (nmtui & hostnamectl)
    - update if needed (dnf update)

    - install build-essentials & make
```bash
dnf install make
dnf install
```

    - Install docker or podman (if podman is installed, you can choses to deploy podman-docker too in order to have same command alias available between docker & podman)
```bash
dnf install podman
dnf install epel-release -y
dnf install podman-compose
dnf install podman-docker
systemctl enable podman
systemctl start podman
chmod o+rw /var/run/podman/podman.sock
```

You’ll need to open up ports 3000 and 7007 to make sure the Backstage app is accessible  --> TCP IN ALLOW



__NODE.JS__

    - install nvm in order to install NPM after --> https://github.com/nvm-sh/nvm#installing-and-updating & https://github.com/nvm-sh/nvm#usage
```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash # (exit terminal session for the nvm install to be effective)
nvm install node # "node" is an alias for the latest version
nvm install v22.18 # install a LTS version - not working with 22.19LTS on 2025-09-16


```

    - install yarn
```bash
npm install -g yarn
npm install -g corepack
yarn init -2
yarn set version stable
yarn install
```

    - install react
```bash
npm install react@latest react-dom@latest
```

__BACKSTAGE__

### Let's now install Backstage using NPM / NPX:

```bash
npx @backstage/create-app@latest
? Enter a name for the app [required] backstage
```

Example of output:
```bash
Creating the app...

 Checking if the directory is available:
  checking      backstage ✔

 Creating a temporary app directory:

 Preparing files:
  copying       .dockerignore ✔
  copying       .eslintignore ✔
  templating    .eslintrc.js.hbs ✔
  templating    .gitignore.hbs ✔
  copying       .prettierignore ✔
  templating    .yarnrc.yml.hbs ✔
  copying       README.md ✔
  copying       app-config.local.yaml ✔
  templating    app-config.yaml.hbs ✔
  templating    backstage.json.hbs ✔
  templating    catalog-info.yaml.hbs ✔
  templating    package.json.hbs ✔
  copying       app-config.production.yaml ✔
  copying       playwright.config.ts ✔
  copying       tsconfig.json ✔
  copying       yarn.lock ✔
  copying       README.md ✔
  copying       yarn-4.4.1.cjs ✔
  copying       entities.yaml ✔
  copying       org.yaml ✔
  copying       template.yaml ✔
  copying       catalog-info.yaml ✔
  copying       index.js ✔
  copying       package.json ✔
  copying       README.md ✔
  templating    .eslintrc.js.hbs ✔
  copying       README.md ✔
  templating    package.json.hbs ✔
  copying       Dockerfile ✔
  copying       index.ts ✔
  templating    .eslintrc.js.hbs ✔
  templating    package.json.hbs ✔
  copying       .eslintignore ✔
  copying       app.test.ts ✔
  copying       android-chrome-192x192.png ✔
  copying       apple-touch-icon.png ✔
  copying       favicon-16x16.png ✔
  copying       favicon-32x32.png ✔
  copying       favicon.ico ✔
  copying       index.html ✔
  copying       manifest.json ✔
  copying       robots.txt ✔
  copying       safari-pinned-tab.svg ✔
  copying       App.test.tsx ✔
  copying       App.tsx ✔
  copying       apis.ts ✔
  copying       index.tsx ✔
  copying       setupTests.ts ✔
  copying       LogoFull.tsx ✔
  copying       LogoIcon.tsx ✔
  copying       Root.tsx ✔
  copying       index.ts ✔
  copying       SearchPage.tsx ✔
  copying       EntityPage.tsx ✔

 Moving to final location:
  moving        backstage ✔
  fetching      yarn.lock seed ✔

 Installing dependencies:
  executing     yarn install ✔
  executing     yarn tsc ✔

🥇  Successfully created backstage


 All set! Now you might want to:
  Run the app: cd backstage && yarn start
  Set up the software catalog: https://backstage.io/docs/features/software-catalog/configuration
  Add authentication: https://backstage.io/docs/auth/
```

Then some adapattions are needed in order to move from a pure proto to something usable.

Change into the backstage folder by typing "cd backstage", and use your preferred editor to edit the following configurations:

```bash
cd backstage
vi app-config.yaml
```

#### Modify app-config.yaml in order to "expose" the app outside "localhost"

"**Key** : Value"\
**app.baseUrl** : http://0.0.0.0:3000/\
**backend.baseUrl** : http://<ip_address_of_your_vm>:7007\
**backend.cors.origin** : http://<ip_address_of_your_vm>:3000

\--

#### Connect the Backstage app to a statefull postgresql database:

Deploy a PGSql database using podman-compose (or helm & values in K8S --> dedicated chapter to follow)
- create docker-compose.yaml with associated content (it will deploy a pgSQL + adminer + pgadmin containers)
```bash
vi docker-compose.yaml
```
```bash
# Use postgres/example user/password credentials
version: "3.9"
services:
  postgres:
    image: "postgres"
    ports:
      - "5432:5432"
    environment:
      POSTGRES_PASSWORD: passwd
      POSTGRES_USER: backstage 
      POSTGRES_DB: backstage
  adminer:
    image: adminer
    ports:
      - 8080:8080
  pgadmin:
    image: dpage/pgadmin4
    ports:
      - 5050:8081
    environment:
       PGADMIN_DEFAULT_EMAIL: "pgadmin@zbook.lan"
       PGADMIN_DEFAULT_PASSWORD: "passwd"
```
- then launch the PGSql compose application:
```bash
podman-compose up -d
```

To enable Backstage to connect to the PostgreSQL database, you need to install the backend package for PostgreSQL. From the backstage folder, run the following:

```bash
yarn add -cwd packages/backend pg --ignore-workspace-root-check
```

Then, in the app-config.yaml file, you need to modify the database section.

In the database section, find this code:
```bash
backend:
  database:
    client: better-sqlite3
    connection: ':memory:'
```
an replace by (or comment before addind)
```bash
backend:
  database:
    client: pg
    connection:
      host: 127.0.0.1   # Because you're running locally
      port: 5432        # The standard PostgreSQL port (could be 5433)
      user: backstage   # The user you created earlier
      password: passwd  # The password for the 'backstage' user
```

__ADD GITHUB AUTHANTICATION__

follow this official tuto works : https://backstage.io/docs/getting-started/config/authentication

Go to https://github.com/settings/applications/new to create your OAuth App. The "Homepage URL" should point to Backstage's frontend, in our tutorial it would be http://IP-of-your-VM:3000. The "Authorization callback URL" will point to the auth backend, which will most likely be http://IP-of-your-VM:7007/api/auth/github/handler/frame.

Take note of the Client ID and the Client Secret (clicking the "Generate a new client secret" button will get this value for you). Open app-config.yaml, and add them as clientId and clientSecret in this file. It should end up looking like this:
app-config.yaml

```bash
auth:
  # see https://backstage.io/docs/auth/ to learn about auth providers
experimentalExtraAllowedOrigins: [ 'http://ip_address_of_your_vm:3000' ]  # because we use local IPs & not private ones
environment: development
  providers:
    # See https://backstage.io/docs/auth/guest/provider
    guest: {}
    github:
      development:
        clientId: YOUR CLIENT ID
        clientSecret: YOUR CLIENT SECRET
```

The next step is to change the sign-in page. For this, you'll actually need to write some code.

Open packages/app/src/App.tsx and below the last import line, add:
```bash
import { githubAuthApiRef } from '@backstage/core-plugin-api';
```

Search for const app = createApp({ in this file, and replace:
```bash
components: {
  SignInPage: props => <SignInPage {...props} auto providers={['guest']} />,
},
```
with
```bash
components: {
  SignInPage: props => (
    <SignInPage
      {...props}
      auto
      provider={{
        id: 'github-auth-provider',
        title: 'GitHub',
        message: 'Sign in using GitHub',
        apiRef: githubAuthApiRef,
      }}
    />
  ),
},
```

Next we need to add the sign-in resolver to our configuration. Here's how:  app-config.yaml

```bash
auth:
  # see https://backstage.io/docs/auth/ to learn about auth providers
  environment: development
  providers:
    # See https://backstage.io/docs/auth/guest/provider
    guest: {}
    github:
      development:
        clientId: YOUR CLIENT ID
        clientSecret: YOUR CLIENT SECRET
        signIn:
          resolvers:
            # Matches the GitHub username with the Backstage user entity name.
            # See https://backstage.io/docs/auth/github/provider#resolvers for more resolvers.
            - resolver: usernameMatchingUserEntityName
```

What this will do is take the user details provided by the auth provider and match that against a User in the Catalog. In this case - usernameMatchingUserEntityName - will match the GitHub user name with the metadata.name value of a User in the Catalog, if none is found you will get an "Failed to sign-in, unable to resolve user identity" message. We'll cover this in the next few sections.

\
Next step is to add auth provider to backstage backend: we will first need to install the package by running this command
from your Backstage root directory:

```bash
yarn --cwd packages/backend add @backstage/plugin-auth-backend-module-github-provider # does not work as of 2025-09-16
```

Then we will need to add this line: in packages/backend/src/index.ts
```bash
backend.add(import('@backstage/plugin-auth-backend'));
backend.add(import('@backstage/plugin-auth-backend-module-github-provider'));
```
Restart Backstage from the terminal, by stopping it with Ctrl+C, and starting it with yarn start. You should be welcomed by a login prompt! If you try to login at this point you will get a "Failed to sign-in, unable to resolve user identity" message, read on as we'll fix that next.


**WARNING:**
A ce stade, le build de Backstage part en vrille avec des erreurs d'intégration à GitHub.


Un update de Backstage en utilisant Backstage-CLI remonte également des erreurs.
