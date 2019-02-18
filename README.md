# Deploying Jekyll on Platformsh

*date: 2019-02-17T18:39:10+01:00*

## 1. Setup your local machine

You'll need 3 tools to deploy your Jekyll site on Platform.sh:

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

# The hooks that will be triggered when the package is deployed.
hooks:
    # Build hooks can modify the application files on disk but not access any services like databases.
    build: |
        bundle install
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

This file tells the platform router to direct all incoming requests to our `jekyll` container.

The last step is to create the appropriate `Gemfile` for our dependencies:

```yaml
source "https://rubygems.org"
gem "jekyll", "~> 3.8.5"
gem "minima", "~> 2.0"
group :jekyll_plugins do
  gem "jekyll-feed", "~> 0.6"
end
gem "tzinfo-data", platforms: [:mingw, :mswin, :x64_mingw, :jruby]
gem "wdm", "~> 0.1.0" if Gem.win_platform?
```

 Commit these new files:

```sh
git add .platform.app.yaml .platform Gemfile
git commit -m "Add platform.sh configuration"
```

## 5. Test and deploy

We are now ready to deploy the project on Platform.sh. Push the repository to the new remote:

```sh
git push platform master
```

Let's review the ouput of your `push`. The first part is basic git. Files and commits are getting pushed to the remote:

```sh
Counting objects: 3, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 613 bytes | 613.00 KiB/s, done.
Total 3 (delta 2), reused 0 (delta 0)
```

Platform.sh then analyze your repository, pull submodules and validate your configuration syntax:

```sh
Validating submodules

Validating configuration files
```

The most important part is now to build your container image (the `build` hook). It will execute the `npm run build` we defined. As there is a `package.json` file in the folder, Platform.sh also launch an `install`

```sh
Processing activity: Guillaume Moigneu pushed to Master
    Found 4 new commits

    Building application 'jekyll' (runtime type: ruby:2.5, tree: db94bb7)
      Generating runtime configuration.
      
      Executing build hook...
        The dependency tzinfo-data (>= 0) will be unused by any of the platforms Bundler is installing for. Bundler is installing for ruby but the dependency is only for x86-mingw32, x86-mswin32, x64-mingw32, java. To add those platforms to the bundle, run `bundle lock --add-platform x86-mingw32 x86-mswin32 x64-mingw32 java`.
        Fetching gem metadata from https://rubygems.org/...........
        Fetching public_suffix 3.0.3
        Installing public_suffix 3.0.3
        ...
        Configuration file: /app/_config.yml
                    Source: /app
               Destination: /app/_site
         Incremental build: disabled. Enable with --incremental
              Generating... 
               Jekyll Feed: Generating feed for posts
                            done in 0.424 seconds.
         Auto-regeneration: disabled. Use --watch to enable.
      
      Executing pre-flight checks...

      Compressing application.
      Beaming package to its final destination.
```

Platform.sh then checks that everything seems correct and deploys the container to a host. You'll see that Platform.sh also generates the Let's Encrypt TLS certificate for your project. 

```sh
Provisioning certificates
Environment certificates
- certificate 43e7fdd: expiring on 2019-05-19 13:12:37+00:00, covering master-7rqtwti-<project ID>.<region>.platformsh.site


Re-deploying environment <project ID>-master-7rqtwti
Environment configuration
  jekyll (type: ruby:2.5, size: S, disk: 5120)

Environment routes
  http://master-7rqtwti-<project ID>.<region>.platformsh.site/ redirects to https://master-7rqtwti-<project ID>.<region>.platformsh.site/
  https://master-7rqtwti-<project ID>.<region>.platformsh.site/ is served by application `jekyll`
```

The last output is the new URL of your application. You can also check that the project has been successully deployed on the web interface:

Now go the URL and you will be able to see your Jekyll site.