# How to create docker images and containers for Rails
## The following steps assume you have downloaded docker desktop and docker-compose 
## Note: if you downloaded docker desktop for mac you do not need to do any additional installation for docker-compose.
## Note: This is just a very basic how-to, I am by no means an expert, but this is how I got my Rails app successfully built in containers and created custom info. 
## It may require some googling for errors. As I understand it, you will not have to build containers from scratch very often. I hope this helps! 
# Steps
## Author: Wyatt Wicks 
1. Create a file named docker-compose.yml within the root of your app, copy/paste the folllowing 
```
# Use the file format compatible with Docker Compose 3.8
version: "3.8"

# Each thing that Docker Compose runs is referred to as
# a "service". In our case, our Rails application is one
# service ("web") and our PostgreSQL database instance
# is another service ("database").
services:
  postgres:
    image: postgres:latest
    environment:
      POSTGRES_USER: your_app_name #DockerHub postgres docs state this is optional but must be used when password is set.  It will also create a db under the supplied username which you'll use to connect to in rails console such as: $docker-compose exec postgres psql -U YourUserNameHere
      POSTGRES_PASSWORD: 'whateverPWYouWant'
    volumes:
      - postgres:/var/lib/postgresql/data 
  web:
    # The root directory from which we're building.
    build: .
    image: dockerhubusername/image_name(probably your appname_web:latest
    links:
      - postgres
    volumes:
      - postgres:/var/lib/postgresql/data 

    # The command to be run when we run "docker-compose up".
    command: bash -c "rm -f tmp/pids/server.pid && bundle exec rails s -p 3000 -b '0.0.0.0'"

    # Expose port 3000, if 3000 is already being used, change to a new port here and in the above command, ex: 3001. 
    ports:
      - "3000:3000"
    env_file:
      - .env_file_name_from_step_3.env
    depends_on:
      - postgres

    # Specify the values of the environment variables
    # used in this container.
    environment:
      RAILS_ENV: development
      DATABASE_NAME: your_app_name_development
      DATABASE_USER: your_app_name_development
      DATABASE_PASSWORD: 'whateverPWYouWant'
      DATABASE_HOST: postgres

# Declare the volumes that our application uses.
volumes:
  postgres: 
```
2. Create a file named Dockerfile within the root of your app, copy/paste the following and change as needed
```
FROM ruby:2.5.3 
# change Ruby version to match what your app uses. 

# throw errors if Gemfile has been modified since Gemfile.lock
RUN bundle config --global frozen 1

WORKDIR /usr/src/app 
#this can be anything

COPY Gemfile Gemfile.lock ./
RUN gem install bundler
#install rake version used for your project as seen in gemlock file.
RUN gem install rake -v 13.0.4
RUN bundle install

COPY . .

CMD ["rails", "server", "-b", "0.0.0.0"]]
```
3. Create .your_app_name.env file in root directory, copy/paste/change. 
 ```
  DATABASE_URL=postgresql://postgresusernamefromstep1:whateverPWYouWant@postgres:5432/app_name_or_postgres_user_name?encoding=utf8&pool=5&timeout=5000
```
4. Go to config file, find database.yml, delete what's in there and copy/paste/change the following, it will look a lot like step 1, but slightly different.
```
default: &default
  adapter: postgresql
  encoding: unicode
  port: 5432
  host: pg
  pool: 5
  timeout: 5000

#This setup works for me running rails on docker. Basically need to specify a URL so you'll use a .env file to specify the postgres url and you'll change between _development _test and _production using your config/database file just make you add the env. file to your web service docker-compose file:

/docker-compose.yml:

services:
  postgres:
    image: postgres:9.6
    restart: always
    environment:
      POSTGRES_USER: same_name_as_in_step_1 #DockerHub postgres docs state this is optional but must be used when password is set.  It will also create a db under the supplied username which you'll use to connect to in rails console such as: $docker-compose exec postgres psql -U YourUserNameHere
      POSTGRES_PASSWORD: 'whateverPWYouWant'
    ports:
      - '5432' #Was originally 5432:5432 with the Left hand side being port on host machine, right hand side is the port on the docker container.  However I let docker choose the port it will use by supplying because 5432 is running on my local for other projects.  
    volumes:
      - postgres:/var/lib/postgresql/data 



  web:
    build: . #Runs the docker build command on the current directory
    links: #Links the listed services to our application so containers can talk to eachother
      - postgres
    restart: always
    volumes:
      - .:/postgres:/var/lib/postgresql/data  #Left hand side is current directory of compose file, right hand side is container folder.  This needs to be same as Install_path folder in Dockerfile.
    ports:
      - '8000:8000' #Left is the local port and the right side is the container port
    env_file:
      - .roomies-be.env #This should be in your root project directory along side the Dockerfile and docker-compose file
    depends_on:
      - 'redis'
      - 'postgres'

development:
  url: <%= ENV['DATABASE_URL'].gsub('?', '_development?' ) %>
test:
  url: <%= ENV['DATABASE_URL'].gsub('?', '_test?' ) %>
production:
  url: <%= ENV['DATABASE_URL'].gsub('?', '_production?' ) %>
  ```
 5. run docker-compose build in terminal
 6. run docker-compose up in terminal, ensure the server runs and you dont get errors.
 6.5 if you get errors you can try docker-compose up --remove-orphans.
 7. close the docker-compose server.
 8. run docker-compose run web bundle exec rake db:create in terminal, ensure DB says created.
 9. run docker-compose run web bundle exec rake db:migrate in terminal, ensure no errors.
 10. run command from step 6 again. Go to localhost://3000 (or whatever port you used if not 3000), it should give you the general rails welcome page.
 11. open docker desktop app, ensure your new containers/images are there.
 12. If you want to push your image to dockerhub, click the 3 dots on the right side of the image name, click push. 
 13. go to dockerhub and see if your image is there.
 
 ## Helpful Articles
 - https://www.codewithjason.com/dockerize-rails-application/
 - https://stackoverflow.com/questions/50090012/how-do-i-run-rails-in-docker-pgconnectionbad-could-not-translate-host-name-p/52179797
