# Developing

You can find the source code on github: [https://github.com/squidex/squidex](https://github.com/squidex/squidex)

## Tools
To work with the source code you need the following tools:

* [Visual Studio Code](https://code.visualstudio.com/) or [Visual Studio 2017](https://www.visualstudio.com/vs/visual-studio-2017-rc/)
* [Node.js](https://nodejs.org/en/)
* [.NET Core SDK](https://www.microsoft.com/net/download/core#/current) (Already part of Visual Studio 2017)
* [MongoDB](https://www.mongodb.com/)
* Optionally: [Redis](https://redis.io/download)
* Optionally: [RabbitMQ](https://www.rabbitmq.com/download.html)
* [Python 2](https://www.python.org/download/releases/2.7/) required for running node sass

We also provide ready to use docker configurations: [https://github.com/squidex/squidex-docker](https://github.com/squidex/squidex-docker). Just execute the following commands to get a mongodb and redis installation for development:

1. `git clone https://github.com/squidex/squidex-docker`
2. `cd squidex-docker/dependencies`
3. `docker-compose up -d`

> Please Note: MongoDB and Redis are not password protected. Do not expose it to the internet.

## How to run the Squidex

The management portal is written with [Angular](https://angular.io) and [Webpack2](https://webpack.js.org/). Therefore you have to run the webpack web dev server which automatically detects changes and builds the application, whenever a file is changed. The typescript code and sass files will be compiled.

The website itself is written in [ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/). For the frontend it just provides a single html file which links to the files from the webpack dev server.

### How to run the Webpack Dev Server?

1. `cd src/Squidex/Squidex` (Go to the web application project)
2. `npm i` (Install all dependencies for the frontend)
3. `npm rebuild node-sass --force`
4. `npm run dev` (Runs the webpack vdev server)
5. Optionally: `npm test` (Runs the unit tests and listens for changes)
6. Optionally: `npm run test:coverage` (Runs the unit tests and calculates the test coverage).

### How to run the Website?

1. `cd src/Squidex/Squidex` (Go to the web application project)
2. `dotnet restore` (Install all dependencies)
3. `dotnet run` (Run the API)

> Open `http://localhost:5000` to run Squidex.

You have to run both indendently. The first time it feels redundant and annoying and we also had some code to run the webpack dev server automatically when the application is started. But you will recognize that it takes a minute for the webpack dev server to start. Therefore we decoupled the commands, so that you can keep the webpack dev server running, even when you have to restart the dotnet application.

As the name 'webpack dev server' indicates, it is only used for development. For production we bundle all typescript files, html and sass and add the bundle to the deployment package.
