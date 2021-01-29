# Deployer live demo

1. Start with API-platform-based project & deployer added as dev dependency
1. Start the project with:

    ```bash
    docker-compose build
    docker-compose up -d
    ```

1. Get into dev container

    ```bash
    docker-compose exec local_dev bash
    ```

    Inside run:

    ```bash
    composer install
    bin/console doctrine:migrations:migrate  # after that check http://localhost:18080 and create few greetings!!!!
    ./vendor/bin/dep init # select symfony, skip repository,
    chmod 666 deploy.php
    ```

1. Open the `deploy.php` file:

    Change recipe path:

    ```php
    require __DIR__ . '/vendor/deployer/deployer/recipe/symfony4.php';
    ```

    **Now is a good moment to exactly see what's happening in the file!**

    Change repository and adjust name:

    ```php
    set('application', 'demo-app');

    // Below is important! We didn't setup any keys to allow cloning via ssh
    set('repository', 'https://github.com/bsacharski/deployer-demo-live.git');
    ```

    Get rid of hosts section
    Get rid of `task`, `after`, `before`

    Create `hosts.yml` file:

    ```yaml
    .base: &base
        forwardAgent: false
        multiplexing: true
        roles:
            - app
        sshOptions:
            UserKnownHostsFile: /dev/null
            StrictHostKeyChecking: no
        keep_releases: 3
        user: deployer
        deploy_path: /var/www/html

    dev:
        <<: *base
        hostname: dev.test
        stage: dev
        branch: master

    prod:
        <<: *base
        hostname: prod.test
        stage: prod
        branch: master
    ```

    add inventory line:

    ```php
    inventory(__DIR__ . '/hosts.yml');
    ```

1. Make first deploy to DEV:

    ```bash
    # switch to Deploy container:
    docker-compose run --rm deployer sh
    # inside run
    ./vendor/bin/dep deploy dev #Stop a while to show the list of ccmmands
    ```

    Failed, whoops! Add following:

    ```php
    // By default the deployer WILL NOT install dev dependencies.
    // We want them in DEV env, so override how composer works
    host('dev')
        ->set('composer_options', '{{composer_action}} --verbose --prefer-dist --no-progress --no-interaction --no-suggest');
    ```

    Deploy again. Whooops again, unlock.

    ```bash
    ./vendor/bin/dep deploy:unlock dev
    ./vendor/bin/dep deploy dev
    ```

    Go to http://localhost:8080/ to see it working, checkout POST endpoint.
    Oh noes, it doesn't run migrations. What to do?

    We're missing config. So let's create a sample one in `.env.dev.local`:

    ```plain
    DATABASE_URL=postgres://api-platform:!ChangeMe!@db_dev/api?server_version=12
    ```

    Now let's add a step do copy the file to the server

    ```php
    task('deploy:config', function () {
        upload('.env.{{stage}}.local', '{{deploy_path}}/shared/.env.local');
    });

    after('deploy:shared', 'deploy:config');
    ```

    Let's check the enpoint once again. Whooops, fail! We don't have schema!

    Add automated migrations and deploy again, check the endpoint

    ```php
    after('deploy:cache:clear', 'database:migrate');
    ```

    Alternatively we could use `database:migrate` command.

1. Let's check what's going on the dev machine - there were some interesting
      paths noted in the deploy log, like `/var/www/html/releases/2`...

      ```bash
      docker-compose exec server_dev bash
      ls -la # Note the symlinks for shared files & current dir, note the + sign at end of ACL
      cd current
      getfacl src
      getfacl var # Note the relation between writable files and getfacl result
      ```

1. Ok, so our system is up and running. Let's add some greetings, then test `GET` endpoint.

1. We're awesome, our system is printing money for our client. O-oh, what is this? A new feature? We need to track when a greeting was created.

    Edit the `src/Entity/Greeting.php` file by adding:

    ```php
    /**
     * @var DateTimeInterface
     * @ORM\Column(type="datetimetz", nullable=false)
     */
    private $createdAt;

    public function getCreatedAt(): DateTimeInterface
    {
        return $this->createdAt;
    }
    ```

    Go back to local dev server console and create migrations:

    ```bash
    bin/console make:migration
    chmod 666 src/Migrations/*
    ```

    Adjust the new migration to have following:

    ```php
    $this->addSql('ALTER TABLE greeting ADD created_at TIMESTAMP(0) WITH TIME ZONE');
    $this->addSql("UPDATE greeting SET created_at = NOW()");
    $this->addSql("ALTER TABLE greeting ALTER COLUMN created_at SET NOT NULL;");
    ```

    Also remove THAT public schema from downgrade!!!

    ```php
    $this->addSql('CREATE SCHEMA public');
    ```

    Run the migration in local dev and verify the migration in **ONLY BY USING GET!!!**

1. Ok, we have new feature, lets commit (**BUT DO NOT PUSH!**)

    With change commited, now let's deploy and see our change.

    Oh noes, the change is not there, but why? I mean I've pushed these changes, right?

    Do a git push, see that only now we pushed the commit to repo. Ok, deploy again!.

1. Ok, we see that making "useless" deploy might be bad. Let's add a security fix in `deploy.php`:

    ```php
    require __DIR__ . '/vendor/deployer/deployer/recipe/deploy/check_remote.php';

    after('deploy:info', 'deploy:check_remote');
    ```

    Now try to deploy once again. Et voila! No more no-commit deploys!

1. Ok, back to our app, let's once again see it. GET yeah, ok, POST...whoops, bug again. What do we do? ROLLBACK

1. Deployer has a `rollback` command built in - what it does it basically changes the symlink to old revision and removes the directory.

    What about the DB though? We could ssh to dev server and run rollback command ourselvers, but why bother?

    ```php
    task('database:rollback', function () {
        $releases = get('releases_list');

        $currentMigrations = run("ls {{deploy_path}}/releases/{$releases[0]}/src/Migrations");
        $prevMigrations = run("ls {{deploy_path}}/releases/{$releases[1]}/src/Migrations");
        if ($prevMigrations !== $currentMigrations) {
            run('{{bin/console}} doctrine:migrations:migrate prev');
        } else {
            writeln('No changes in migration files. Skipping database rollback.');
        }
    })->desc('Rollback database');
    before('rollback', 'database:rollback');
    ```

    Now run:

    ```bash
    ./vendor/bin/dep rollback dev
    ```

    After that, hard refresh the site to verify that GET and POST are working as expected.

1. So we got the system working for time being, now lets bo back to the feature (and fix it).

    Go to local dev and see once again what is happening. Ah, we're not setting the `created_at` value. No problem:

    ```php
    @ORM\HasLifecycleCallbacks() // class annotation!!!



    /**
     * @ORM\PrePersist()
     * @return void
     */
    public function prePersistHandler()
    {
        $this->createdAt = new DateTimeImmutable();
    }
    ```

    Now test in in local dev server, to be sure it's working.

    Once it's ok, commit it, push it and call deploy to dev again.

    It if works then good job, we delivered a new feature!

1. Now the only thing left to do is to deploy to PROD env now (see http://localhost:8081/).

    We got the hosts ready, but we need to set up the env variables.
    Create the `.env.prod.local` file with following:

    ```plain
    APP_ENV=prod
    DATABASE_URL=postgres://api-platform:!ChangeMe!@db_prod/api?server_version=12
    ```

    Then just run:

    ```bash
    ./vendor/bin/dep deploy prod
    ```

    After deploy, visit http://localhost:8081/ once again to verify that app
    has been deployed in PROD mode (notice the toolbar on bottom is missing).

1. That's it. Now if you want to cleanup, run

    ```bash
    docker-compose down  --volumes --remove-orphans
    ```
