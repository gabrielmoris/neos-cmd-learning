# Setup

1. `docker run -v <MY_LOCAL_ABSOLUTE_PATH>:/app -it --rm composer create-project neos/neos-base-distribution neos-example`
2. Create `Dockerfile.dev`

```dockerfile
FROM php:8.1-cli

RUN apt-get update \
    # install GraphicsMagick
	&& apt-get install -y \
		libgraphicsmagick1-dev graphicsmagick zlib1g-dev libicu-dev gcc g++ --no-install-recommends \
	&& pecl -vvv install gmagick-beta && docker-php-ext-enable gmagick \
    # pdo_mysql
    && docker-php-ext-install pdo_mysql \
    # redis
    && pecl install redis && docker-php-ext-enable redis \
	# intl
	&& docker-php-ext-configure intl && docker-php-ext-install intl \
    # cleanup
    && apt-get clean && rm -rf /var/lib/apt/lists/*


WORKDIR /app
EXPOSE 8081

# copy everything in the project into the container. This is what
# makes this image so fast!
COPY . /app

# start the dev server
CMD [ "./flow", "server:run", "--host", "0.0.0.0" ]
```

3. Create `docker-compose.yml`

```yml
# NEOS DEVELOPMENT ENVIRONMENT
#
# For instructions how to use docker-compose, see
# https://docs.neos.io/cms/installation-development-setup/docker-and-docker-compose-setup#docker-compose-cheat-sheet
version: "3.7"
services:
  # Neos CMS
  neos:
    build:
      context: .
      dockerfile: Dockerfile.dev
    environment:
      FLOW_CONTEXT: "Development/Docker"
    volumes:
      - ./composer.json:/app/composer.json
      - ./composer.lock:/app/composer.lock
      - ./Configuration/:/app/Configuration/
      - ./DistributionPackages/:/app/DistributionPackages/
      # if you work on other packages, you need to add them here.

      # WARNING: you need to add all packages from Distribution packages here ONE BY ONE, see the notice below for explanation.
      - ./Packages/Sites/:/app/Packages/Sites/
    ports:
      - 8081:8081
  # DB
  db:
    image: mariadb:10.7
    environment:
      MYSQL_ROOT_PASSWORD: "db"
    volumes:
      - db:/var/lib/mysql
    ports:
      - 13306:3306

volumes:
  db:
```

4. `docker compose build` and then `docker compose up`
5. Create folder **/Docker** inside `Configuration\Development`
6. The app shuld be reacchable in [http://127.0.0.1:8081/](http://127.0.0.1:8081/)
7. Time to setup evetyrhing: `docker compose exec neos /bin/bash`

   - ./flow setup
   - ./flow setup:database

   ```text
   host: db
   user: root
   password: db
   database: db
   ```

   - ./flow setup:imagehandler
   - ./flow doctrine:migrate

   - ./flow user:create --roles Administrator (makes the password for the admin user)
   - ./flow site:import --package-key Neos.Demo

Happy Codding!

# Implementing React App through an Iframe

1. Create a new directory called react-app within your Neos project directory (next to your Packages, Configuration, etc.) and run `npx create-react-app .`

2. Add the following to your existing docker-compose.yml file

```yaml
services:
  # ... (existing services)
  react:
    build:
      context: ./react-app
      dockerfile: Dockerfile.react
    ports:
      - 3000:3000
```

3. Create a new file named Dockerfile.react _inside_ the react-app directory:

```Dockerfile
# Dockerfile.react
FROM node:14
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm install
COPY . .
EXPOSE 3000
CMD ["npm", "start"]

```

4. Integrate React Component in Neos
   - Create in `http://localhost:8081/neos` a new page /react
   - After using docker compose up the `http://localhost:3000` should run the react app, That means I can create an HTML component and implement an Iframe with this code:
   ```html
   <style>
     body,
     html {
       margin: 0;
       padding: 0;
       height: 100%;
     }
     iframe {
       display: block;
       width: 100%;
       height: 100%;
       border: none;
     }
   </style>
   <body>
     <iframe src="http://localhost:3000" frameborder="0"></iframe>
   </body>
   ```
