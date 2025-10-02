
# 3env-sf-gen-ci

![Symfony](https://img.shields.io/badge/Symfony-5.x-000000?style=flat&logo=symfony)
![PHP](https://img.shields.io/badge/PHP-7.4+-777BB4?style=flat&logo=php)
![Docker](https://img.shields.io/badge/Docker-enabled-2496ED?style=flat&logo=docker)

## Description

**3env-sf-gen-ci** is a pre-configured Symfony project template with a complete CI/CD infrastructure for gitlab CI. This project is designed to be used in conjunction with the reference template **[sf-gen-ci-ref](../sf-gen-ci-ref)** which contains all Docker configuration, deployment scripts, and GitLab CI/CD pipelines.

## Purpose

This template demonstrates the implementation of a Symfony application with:
- Automated CI/CD configuration (GitLab CI)
- Unit and functional tests
- Code quality analysis (SonarQube)
- Multi-environment deployment (dev, staging, production)
- Demo CRUD with fixtures
- Complete Docker containerization

## Architecture

This project is part of a two-component system:

1. **3env-sf-gen-ci** (this project): Symfony Application
2. **[sf-gen-ci-ref](../sf-gen-ci-ref)**: DevOps Infrastructure and Configuration
```

edouardpeyrot/
├── 3env-sf-gen-ci/        # Symfony Application
│   ├── src/                    # Source code
│   ├── tests/                  # PHPUnit tests
│   ├── .gitlab-ci.yml          # CI/CD Pipeline
│   └── composer.json
└── sf-gen-ci-ref/        # DevOps Infrastructure
    ├── docker/                 # Docker configuration
    ├── backup/                 # Backup scripts
    └── .gitlab-template.yml    # Reusable CI/CD template
```

## Quick Start

### Prerequisites

- PHP 7.4+
- Composer
- Docker & Docker Compose (for deployment)
- Git

## 1. Setup a project

Here is a project example template for CI/CD. 

Don't directly use this project to develop. Dependencies might not be up to date.

### 1.1 Download framework
    
```bash 
    symfony new myproject
    #Then, add /.idea & /.vscode in your .gitignore
    
    # To use this template install following dependencies
    composer req doctrine/annotations doctrine/doctrine-bundle \
    doctrine/doctrine-migrations-bundle doctrine/orm symfony/asset \
    symfony/console symfony/dotenv symfony/expression-language symfony/flex \
    symfony/form symfony/framework-bundle symfony/http-client symfony/intl \
    symfony/mailer symfony/mime symfony/monolog-bundle symfony/process symfony/runtime \
    symfony/security-bundle symfony/twig-bundle symfony/yaml twig/extra-bundle twig/twig
    # Say no to docker-compose (already managed by sf-gen-ci-ref)
    composer req --dev doctrine/doctrine-fixtures-bundle symfony/browser-kit symfony/css-selector \ 
    symfony/debug-bundle symfony/maker-bundle symfony/phpunit-bridge symfony/stopwatch \ 
    symfony/var-dumper symfony/web-profiler-bundle
    
    
    git branch develop 
    git checkout develop
    git add . 
    git commit -m 'first commit'
    git push origin develop

    
```

### 1.2 Configure repo
 - Create a repository on GitLab
 - Clone the repository on your computer
 - Add the remote repository to your local repository
 - Create a branch named develop

### 1.3 Configure CI/CD

 - Copy the .gitlab.yml file and replace db name, db password. 
 - Commit/push on develop
 - Copy docker folder and fill it to your project

### 1.4 Configure environment variables

DOMAIN_NAME : The domain name (ex: www.example.com). Note the CI will automatically add the -preprod suffix for pre-production environment.

MYSQL_HOST_PORT : The mysql port (on host)

WWW_HOST_PORT : The apache port (on host)

Note routing with another reverse proxy like traefik is not supported.

### 1.5 Apache Server

This project uses Apache2 as web server into the docker container. It also uses apache2 on host as a reverse proxy.

See apache2/ folder for more information in the sf-gen-ci-ref repository.

### Access the Application

- **Application**: http://localhost:8000
- **Demo CRUD**: http://localhost:8000/demo

## Fixtures

The project includes demo fixtures to quickly test the application:

```shell script
# Load fixtures (replaces existing data)
php bin/console doctrine:fixtures:load

# Load fixtures in append mode (adds without deleting)
php bin/console doctrine:fixtures:load --append
```


**Fixtures file**: `src/DataFixtures/AppFixtures.php`

Fixtures generate sample data for the `Demo` entity to test the CRUD operations.

## Project Structure

```
src/
├── Controller/         # Symfony Controllers
├── Entity/            # Doctrine Entities
│   └── Demo.php       # Demo entity
├── Form/              # Symfony Forms
│   └── DemoType.php   # Form for Demo entity
├── Repository/        # Doctrine Repositories
└── DataFixtures/      # Fixtures for testing and demo
    └── AppFixtures.php

templates/
├── demo/              # Twig templates for CRUD
└── base.html.twig     # Base template

tests/
├── FunctionalTest.php # Functional tests
└── UnitTest.php       # Unit tests
```


## CI/CD Pipeline (GitLab)

The `.gitlab-template.yml` file defines a complete pipeline with **7 stages**:

### Stage 1: Prepare
**Purpose**: Prepare the environment and fetch reference repository

**Actions**:
- Install Git and configure SSL certificates
- Clone the sf-gen-ci-ref reference repository
- Copy Docker configuration to the project
- Set up Git credentials for secure access


**When**: Runs on every commit

---

### Stage 2: Docker
**Purpose**: Build Docker images for the application

**Actions**:
- Build the PHP/Apache Docker image with custom configuration
- Build the SonarQube scanner image for code quality analysis
- Tag images with appropriate versions
- Test that the built image works correctly

**Key Commands**:
```yaml
- docker compose build --build-arg NODE_VERSION=$NODE_VERSION
- docker build -f php/Dockerfile -t www_${CI_PROJECT_NAME}:latest .
- docker build -f sonarqube/Dockerfile -t sonarscanner:latest .
```


**Parallel Jobs**:
- `docker`: Build images
- `test:docker_www`: Verify PHP version in built image

**When**: Runs after prepare stage

---

### Stage 3: Build
**Purpose**: Build the application and install dependencies

**Actions**:
- Install Composer dependencies
- Install NPM dependencies (if package.json exists)
- Build frontend assets (if applicable)
- Generate environment configuration
- Configure Apache virtual hosts
- Copy application to web server directory
- Restart Apache service

**Key Commands**:
```yaml
- composer install
- npm install && npm run build
- ./docker/generate_env.sh
- systemctl restart apache2
```


**Environment Variables**:
- `APP_ENV=dev`
- `APP_DEBUG=true`
- Database configuration via MySQL service

**Services Used**:
- MySQL 5.7 for database operations

**When**: Runs after Docker stage completes

---

### Stage 4: Tests
**Purpose**: Execute automated tests

**Job: phpunit**

**Actions**:
- Set up test environment with MySQL service
- Install PHP extensions (mysqli, pdo, zip)
- Create and configure test database
- Load fixtures into test database
- Run PHPUnit test suite

**Key Commands**:
```yaml
- php bin/console doctrine:database:create --env=test
- php bin/console doctrine:schema:update --force --env=test
- php bin/console doctrine:fixtures:load --env=test
- php bin/phpunit
```


**Environment Variables**:
- `PIPELINE_ENV=test`
- `DATABASE_URL=mysql://root:${MYSQL_ROOT_PASSWORD}@mysql:3306/${CI_PROJECT_NAME}`

**When**: Runs only on develop branch
**Failure Handling**: Pipeline stops if tests fail

---

### Stage 5: Quality
**Purpose**: Perform code quality checks and security analysis

**Jobs**:

#### 5.1 security-checker
- Scan composer.lock for known security vulnerabilities
- Uses local-php-security-checker tool
- **Blocking**: Pipeline fails if vulnerabilities found

#### 5.2 php-coding-standards
- Check PSR-12 coding standards compliance
- Auto-fix code formatting issues
- Verify code style (excluding Kernel.php)
- **Non-blocking**: Warnings only

**Commands**:
```yaml
- phpcbf --standard=PSR12 ./src
- phpcs -v --standard=PSR12 --ignore=./src/Kernel.php ./src
```


#### 5.3 phpstan
- Static code analysis for type errors and bugs
- Analyzes src/ directory
- Requires LDAP extensions installation
- **Non-blocking**: Warnings only

**Commands**:
```yaml
- phpstan analyse ./src
```


#### 5.4 twig-lint
- Validate Twig template syntax
- Ensures no template errors exist
- **Blocking**: Pipeline fails if template errors found

**Commands**:
```yaml
- twig-lint lint ./templates
```


#### 5.5 sonar
- Comprehensive code quality analysis with SonarQube
- Analyzes code smells, bugs, security vulnerabilities
- Generates detailed metrics and reports
- Imports custom SSL certificates for secure connection

**Key Configuration**:
```yaml
SONAR_PROJECT_KEY: "$CI_PROJECT_NAME"
SONAR_SCANNER_OPTS: "-Djavax.net.ssl.trustStore=sonarqube.jks"
```


**When**: All quality jobs run only on develop branch

---

### Stage 6: Deploy (Pre-production)
**Purpose**: Deploy application to pre-production environment

**Job: deploy-preprod**

**Actions**:
- Configure SSH access to pre-production server
- Generate environment-specific configuration
- Create deployment archive (tar.gz)
- Transfer files to server via SCP
- Execute deployment script on remote server
- Start Docker containers
- Create/update database schema
- Load initial data (if needed)
- Clear application cache

**Key Commands**:
```yaml
- tar -zcf $CI_COMMIT_SHA.tar.gz ./* .env .env.test
- scp ${CI_COMMIT_SHA}.tar.gz ${SSH_USER}@${PREPROD_SSH_HOST}:/var/www/
- ssh ${SSH_USER}@${PREPROD_SSH_HOST} "bash deploy.sh"
- docker exec www_${CI_PROJECT_NAME} php bin/console doctrine:database:create
- docker exec www_${CI_PROJECT_NAME} php bin/console doctrine:schema:update --force
```


**Environment Variables**:
- `APP_ENV=dev`
- `APP_DEBUG=true`
- Domain: `${DOMAIN_NAME}-preprod`

**When**: Manual trigger only on develop branch
**SSH Authentication**: Uses PREPROD_SSH_PRIVATE_KEY

---

### Stage 7: Deploy (Production)
**Purpose**: Deploy application to production environment

**Job: deploy:prod**

**Actions**:
- Configure production SSH access
- Generate production environment configuration
- Build with production optimizations
- Create deployment archive
- Transfer to production server
- Execute deployment with zero-downtime strategy
- Update database schema (safe mode)
- Clear production cache

**Key Commands**:
```yaml
- export APP_ENV=prod && export APP_DEBUG=false
- bash ./docker/generate_env.sh
- tar -zcf $CI_COMMIT_SHA.tar.gz ./* .env .env.test
- scp ${CI_COMMIT_SHA}.tar.gz ${SSH_USER}@${PROD_SSH_HOST}:/var/www/
- ssh ${SSH_USER}@${PROD_SSH_HOST} "bash deploy.sh"
```


**Environment Variables**:
- `APP_ENV=prod`
- `APP_DEBUG=false`
- Domain: `${DOMAIN_NAME}`

**Safety Features**:
- Manual approval required
- Only runs on main branch
- Database backup before deployment
- Rollback capability

**When**: Manual trigger only on main branch
**SSH Authentication**: Uses PROD_SSH_PRIVATE_KEY

---

### Pipeline Workflow

```
Commit → Prepare → Docker → Build → Tests → Quality → Deploy Preprod → Deploy Prod
                                      ↓
                                    FAIL → Pipeline stopped
                                      ↓
                                    PASS → Continue
```


### Key Features

- **Automated Testing**: Every commit runs tests
- **Code Quality Gates**: Multiple quality checks before deployment
- **Multi-Environment**: Separate configs for dev, preprod, and production
- **Manual Gates**: Production deploys require manual approval
- **Security**: SSH key-based authentication, vulnerability scanning
- **Zero Downtime**: Docker-based rolling deployments
- **Database Safety**: Automatic schema updates with backups

## Tests

```shell script
# Run all tests
php bin/phpunit

# Run tests with coverage
php bin/phpunit --coverage-html coverage/

# Run specific tests
php bin/phpunit tests/UnitTest.php
php bin/phpunit tests/FunctionalTest.php
```


## Docker Deployment

To deploy the application with the Docker configuration from **sf-gen-ci-ref**:

```shell script
# Navigate to the reference folder
cd ../sf-gen-ci-ref/docker

# Generate .env file
./generate_env.sh

# Start containers
docker-compose up -d

# View logs
docker-compose logs -f

# Stop containers
docker-compose down
```


See complete documentation in [sf-gen-ci-ref/README.md](../sf-gen-ci-ref/README.md)

## Technologies

- **Framework**: Symfony 6.x
- **PHP**: 8.1+
- **Database**: MySQL/PostgreSQL
- **ORM**: Doctrine
- **Testing**: PHPUnit
- **Quality**: SonarQube, PHP-CS-Fixer, PHPStan
- **CI/CD**: GitLab CI
- **Containerization**: Docker, Docker Compose
- **Web Server**: Apache2

## Additional Documentation

- [Docker Configuration](../sf-gen-ci-ref/docker/README.md)
- [Backup Scripts](../sf-gen-ci-ref/backup/README.md)
- [Container Cleanup](../sf-gen-ci-ref/clear_containers/README.md)
- [Official Symfony Documentation](https://symfony.com/doc/current/index.html)

## Contributing

This project is a demonstration template. To adapt it to your project:

1. Clone the repository
2. Modify namespaces and project names
3. Adapt entities to your needs
4. Configure environment variables
5. Adapt the CI/CD pipeline according to your infrastructure

## License

See license file in the root directory.

## Author

Edouard PEYROT - [edouardpeyrot.com](https://edouardpeyrot.com)

---