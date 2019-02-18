# Deploying Jekyll on Platformsh

*date: 2019-02-17T18:39:10+01:00*

## 1. Setup your local machine

You'll need 3 tools to deploy your Hugo site on Platform.sh:

- ruby and rubygem: [install git](https://nodejs.org/en/)
- jekyll: `sudo gem install bundler jekyll`
- Optional: [the platform.sh cli tool](https://docs.platform.sh/gettingstarted/cli.html)

For each of these tools please refer to their own install page.

On macOS, this is basically 4 commands:

```sh
brew install ruby
sudo gem install bundler jekyll
brew tap homebrew/homebrew-php
brew install curl git php72-cli php72-curl
curl -sS https://platform.sh/cli/installer | php
```

## 2. Bootstrap your Jekyll project

We need to create a new Jekyll folder from the command-line:

```sh
$ jekyll new jekyll-hello     
Running bundle install in /media/psh/customers/jekyll-hello... 
Your user account isn't allowed to install to the system RubyGems.
  You can cancel this installation and run:

      bundle install --path vendor/bundle

  to install the gems into ./vendor/bundle/, or you can enter your password
  and install the bundled gems to RubyGems using sudo.

  Password: 
  Bundler: Fetching gem metadata from https://rubygems.org/...........
  Bundler: Fetching gem metadata from https://rubygems.org/.
  Bundler: Resolving dependencies...
  Bundler: Fetching public_suffix 3.0.3
  ...
  Bundler: /var/lib/gems/2.5.0/specifications
New jekyll site installed in jekyll-hello.
```

You can now run the development server:

```sh
$ cd jekyll-hello && bundle exec jekyll serve
Configuration file: jekyll-hello/_config.yml
            Source: jekyll-hello
       Destination: jekyll-hello/_site
 Incremental build: disabled. Enable with --incremental
      Generating... 
       Jekyll Feed: Generating feed for posts
                    done in 0.288 seconds.
 Auto-regeneration: enabled for 'jekyll-hello'
    Server address: http://127.0.0.1:4000/
  Server running... press ctrl-c to stop.
```

Browse [http://localhost:4000/](http://localhost:4000/) to check that everything is working as expected.

The `npx` template already creates a basic git repository:

Create a new git repository inside that folder:

```sh
git init .
git add .
git commit -m "Init jekyll"
```

## 3. Create your platformsh project

Create a Platform.sh project by signing to [a trial account](https://accounts.platform.sh/platform/trial/general/setup) or log in to your account.

Select a `Standard` project and then choose the region you want your project to be hosted in.

Review and validate. You can now access your newly provisioned project. On the wizard, click `Git remote` and copy it.

Add the remote to your local project:

```sh
git remote add platform <project ID>@git.<region>.platform.sh:<project ID>.git
```

**Don't push anything for now**

## 4. Setup the .platform.app.yaml configuration

Platform.sh relies on `yaml` configurations to configure the different containers to deploy. 
Create the `.platform.app.yaml` file at the root of your project and add the following code:

```yaml
# .platform.app.yaml

# The name of this application, which must be unique within a project.
name: 'jekyll'

# The type key specifies the language and version for your application.
type: 'ruby:2.5'

dependencies:
    ruby: # Specify one Bundler package per line.
        jekyll: "*"

# The hooks that will be triggered when the package is deployed.
hooks:
    # Build hooks can modify the application files on disk but not access any services like databases.
    build: |
      bundle exec jekyll build

# The size of the persistent disk of the application (in MB).
disk: 5120

# The configuration of the application when it is exposed to the web.
web:
    locations:
        '/':
            # The public directory of the application relative to its root.
            root: '_site'
            index: ['index.html']
            scripts: false
            allow: true
```

We need also two other files: `routes.yaml` and `services.yaml`. `services.yaml` is used to configure additional services like databases so we don't need it for that project. Just create the file:

```sh
mkdir .platform
touch services.yaml
```

Add `routes.yaml` in the `.platform` folder and add the following configuration:

```yaml
"https://{default}/":
  type: upstream
  upstream: "jekyll:http"
```

This file tells the platform router to direct all incoming requests to our `jekyll` container. Commit these new files:

```sh
git add .platform.app.yaml .platform
git commit -m "Add platform.sh configuration"
```

## 5. Test and deploy

We are now ready to deploy the project on Platform.sh. Push the repository to the new remote:

```sh
git push platform master
```

Let's review the ouput of your `push`. The first part is basic git. Files and commits are getting pushed to the remote:

```sh
Counting objects: 44, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (38/38), done.
Writing objects: 100% (44/44), 1.04 MiB | 16.98 MiB/s, done.
Total 44 (delta 2), reused 0 (delta 0)
```

Platform.sh then analyze your repository, pull submodules and validate your configuration syntax:

```sh
Validating submodules

Validating configuration files
```

The most important part is now to build your container image (the `build` hook). It will execute the `npm run build` we defined. As there is a `package.json` file in the folder, Platform.sh also launch an `install`

```sh
Building application 'jekyll' (runtime type: nodejs:8.9, tree: fafb3d4)
      Generating runtime configuration.
      
      Building a NodeJS application, let's make it fly.
      Found a `package.json`, installing dependencies.
        
        > sharp@0.21.3 install node_modules/sharp
        > (node install/libvips && node install/dll-copy && prebuild-install) || (node-gyp rebuild && node install/dll-copy)
        ...
        up to date in 14.102s

        success open and validate gatsby-configs — 0.009 s
        success load plugins — 0.325 s
        success onPreInit — 0.895 s
        success delete html and css files from previous builds — 0.096 s
        success initialize cache — 0.016 s
        success copy gatsby files — 0.024 s
        success onPreBootstrap — 0.006 s
        success source and transform nodes — 0.104 s
        success building schema — 0.326 s
        success createPages — 0.056 s
        success createPagesStatefully — 0.032 s
        success onPreExtractQueries — 0.004 s
        success update schema — 0.185 s
        success extract queries from components — 0.134 s
        
        success run graphql queries — 0.694 s — 9/9 13.00 queries/second
        success write out page data — 0.012 s
        success write out redirect data — 0.001 s
        
        
        
        info bootstrap finished - 5.674 s
        
        done generating icons for manifest
        success onPostBootstrap — 0.287 s
        success Building production JavaScript and CSS bundles — 11.784 s
        success Building static HTML for pages — 0.844 s — 7/7 28.27 pages/second
        Generated public/sw.js, which will precache 11 files, totaling 283621 bytes.
        info Done building in 18.685 sec
```

Platform.sh then checks that everything seems correct and deploys the container to a host. You'll see that Platform.sh also generates the Let's Encrypt TLS certificate for your project. 

```sh
      Executing pre-flight checks...

      Compressing application.
      Beaming package to its final destination.

    W: Route '{default}' doesn't map to a domain of the project, mangling the route.

    Provisioning certificates
      Validating 1 new domain
      Provisioned new certificate for 1 domains of this environment
      (Next refresh will be at 2019-04-20 20:19:01+00:00.)
      Environment certificates
      - certificate 18bf626: expiring on 2019-05-18 20:19:01+00:00, covering master-7rqtwti-<project ID>.<region>.platformsh.site


    Creating environment <project ID>-master-7rqtwti
      Environment configuration
        jekyll (type: nodejs:8.9, size: M, disk: 5120)

      Environment routes
        http://master-7rqtwti-<project ID>.<region>.platformsh.site/ redirects to https://master-7rqtwti-<project ID>.<region>.platformsh.site/
        https://master-7rqtwti-<project ID>.<region>.platformsh.site/ is served by application `jekyll`
```

The last output is the new URL of your application. You can also check that the project has been successully deployed on the web interface:

Now go the URL and you will be able to see your Jekyll site