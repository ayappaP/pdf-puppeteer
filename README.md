#### Difficulty: Easy to Medium

### Generate a PDF from HTML using Puppeteer

Puppeteer is a library which basically provides high level Api's that lets us run the browser from Node.js. So we can use it to generate Pdf's and many other things.

### Install the following dependencies using yarn or npm

```
    yarn add dev serverless pupeteer
```

```
    yarn add chrome-aws-lambda pug puppeteer-core serverless-apigw-binary serverless-offline
```

### Serverless.yml file

We use the serverless library to handle our AWS step and also to run our lambda functions locally.
Serverless.yml is a configuration file where we can define our AWS resources and do other fun things. Refer the docs.

Before you begin, [setup your AWS Credentials](https://serverless.com/framework/docs/providers/aws/guide/installation/).

Then create a serverless.yml file and add this

```
    service: pdf-puppeteer

    provider:
        name: aws
        runtime: nodejs8.10
        stage: dev
        profile: default

    plugins:
    - serverless-offline

    functions:
        pdf:
            handler: pdf.pdf

```

serverless-offline is a serverless plugin that lets us run the pdf lambda function locally. More on that later.

### Using `puppeteer-core` and `chrome-aws-lambda` for AWS Lambda

These two libraries do all the magic for us. Since the deployment package for AWS Lambda will go over the size limit, we can't use the full version of `puppeteer`. `Puppeteer` is around 300 MB because it downloads Chromium during `yarn add` before exposing the API. But luckily, `chrome-aws-lambda` ships the chromium binary for serverless environments and puppeteer-core is a version of Puppeteer that is only 2 MB and which can connect to our own Chromium instance of choice. So together with chrome-aws-lambda it provides a “full” puppeteer and is small enough to be deployed.

### Gotcha:

#### Using `pupeeteer` for local development

If you try to run our lambda function locally using `serverless invoke local -f pdf`, it won't work because `puppeteer-core` doesn’t download chromium and `chrome-aws-lambda` is only for AWS lambda, ie not for your local environment. To make it run locally, we need to install the full `puppeteer` as devDependencies, and in the code we need to check if the request comes from serverless-offline, if yes then set the executablePath to your local Chromium.

`./node_modules/puppeteer/.local-chromium/mac-674921/chrome-mac/Chromium.app/Contents/MacOS/Chromium` is the local executablePath for chromium we installed with puppeteer on devDependencies.

`event.isOffline` in `pdf.js` checks if we are running serverless locally and sets the executable path accordingly.

### Pdf.js file

This is the full code inside our lambda function.

```
    "use strict";
    const chromium = require("chrome-aws-lambda");
    var AWS = require("aws-sdk");
    const pug = require("pug");
    const fs = require("fs");
    const path = require("path");

    AWS.config.update({ region: "us-east-1" });
    const s3 = new AWS.S3();

    module.exports.pdf = async (event, context, callBack) => {
        const dummyData = {
            name: "Ayappa Reddy"
        };

        const executablePath = event.isOffline
            ? "./node_modules/puppeteer/.local-chromium/mac-674921/chrome-mac/Chromium.app/Contents/MacOS/Chromium"
            : await chromium.executablePath;

        const template = pug.compileFile("./template.pug");
        const html = template({ ...dummyData });

        let browser = null;

        try {
            browser = await chromium.puppeteer.launch({
                args: chromium.args,
                defaultViewport: chromium.defaultViewport,
                executablePath,
                headless: chromium.headless
            });

            const page = await browser.newPage();

            page.setContent(html);

            const pdf = await page.pdf({
                format: "A4",
                printBackground: true,
                margin: { top: "1cm", right: "1cm", bottom: "1cm", left: "1cm" }
            });

            // TODO: Response with PDF (or error if something went wrong )
            const response = {
                headers: {
                    "Content-type": "application/pdf",
                    "content-disposition": "attachment; filename=test.pdf"
                },
                statusCode: 200,
                body: pdf.toString("base64"),
                isBase64Encoded: true
            };

            const output_filename = `invoice.pdf`;

            const s3Params = {
                Bucket: "pdf-puppeteer",
                Key: `public/pdfs/${output_filename}`,
                Body: pdf,
                ContentType: "application/pdf",
                ServerSideEncryption: "AES256"
            };

            s3.putObject(s3Params, err => {
            if (err) {
                console.log("err", err);
                return callBack(null, { error });
            }
            });

            context.succeed(response);

            // fs.writeFileSync("invoice.pdf", pdf); // Locally
        } catch (error) {
            return context.fail(error);
        } finally {
            if (browser !== null) {
            await browser.close();
            }
        }
    };
```

## Saving the pdf to S3 bucket

We have already created a S3 bucket called "pdf-puppeteer" in our AWS account. All the generated pdfs will be saved under `/pdf-puppeteer/public/pdfs/invoice.pdf` if you run the lambda either locally or in development. Please create a bucket in S3 if you wish so and modify `pdf.js` with the correct bucket name in `s3Params`.

If you would like to see your changes locally without saving to S3 all the time and instead to `invoice.pdf` at the root of the project directory. Modify `pdf.js` like so,

```
    "use strict";
    const chromium = require("chrome-aws-lambda");
    var AWS = require("aws-sdk");
    const pug = require("pug");
    const fs = require("fs");
    const path = require("path");

    module.exports.pdf = async (event, context, callBack) => {
        const dummyData = {
            name: "Ayappa Reddy"
        };

        const executablePath = event.isOffline
            ? "./node_modules/puppeteer/.local-chromium/mac-674921/chrome-mac/Chromium.app/Contents/MacOS/Chromium"
            : await chromium.executablePath;

        const template = pug.compileFile("./template.pug");
        const html = template({ ...dummyData });

        let browser = null;

        try {
            browser = await chromium.puppeteer.launch({
                args: chromium.args,
                defaultViewport: chromium.defaultViewport,
                executablePath,
                headless: chromium.headless
            });

            const page = await browser.newPage();

            page.setContent(html);

            const pdf = await page.pdf({
                format: "A4",
                printBackground: true,
                margin: { top: "1cm", right: "1cm", bottom: "1cm", left: "1cm" }
            });

            // TODO: Response with PDF (or error if something went wrong )
            const response = {
                headers: {
                    "Content-type": "application/pdf",
                    "content-disposition": "attachment; filename=test.pdf"
                },
                statusCode: 200,
                body: pdf.toString("base64"),
                isBase64Encoded: true
            };

            context.succeed(response);

            fs.writeFileSync("invoice.pdf", pdf); // Locally
        } catch (error) {
            return context.fail(error);
        } finally {
            if (browser !== null) {
            await browser.close();
            }
        }
    } ;
```

All we did here was remove the code that saves the pdf to S3 and instead uncomment the part that says `fs.writeFileSync("invoice.pdf", pdf)`

### Creating an HTML template using `pug` for our PDF.

    We use `pug` as the templating engine for our HTML.

    Create a file `template.pug` at the root of the project directory and add your HTML.

    In `pdf.js`, the two lines below will compile our template file with the data, possibly from the db.

```
    const template = pug.compileFile("./template.pug");
    const html = template({ ...dummyData });
```

Here is a basic `template.pug` file.

```
    doctype html
    html(lang='en')
        head
            meta(charset='UTF-8')
            title PDF Generator
            style
            include style.css
        body
            h1 Monthly report

            #body Here comes the values #{name}
```

The line `include style.css` includes our stylesheet `style.css` from the project root.

```
    body {
        font-family: Helvetica;
    }

    h1 {
        font-size: 36px;
        border-bottom: 1px solid red;
    }

    h3 {
        font-size: 16px;
    }


```

Refer the `pug` docs for more info on creating your HTML.

Finally, use `sls deploy` to deploy the lambda function to AWS.
