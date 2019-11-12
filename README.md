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
    html(lang="en")
    head
        meta(charset="utf-8")
        title Example 2
        style
        include style.css
    body
        header.clearfix
        #logo
            img(src="./logo.png")
        #company
            h2.name Company Name
            div 455 Foggy Heights, AZ 85004, US
            div (602) 519-0450
            div
            a(href="mailto:company@example.com") company@example.com
        main
        #details.clearfix
            #client
            .to INVOICE TO:
            h2.name John Doe
            .address 796 Silver Harbour, TX 79273, US
            .email
                a(href="mailto:john@example.com") john@example.com
            #invoice
            h1 INVOICE 3-2-1
            .date Date of Invoice: 01/06/2014
            .date Due Date: 30/06/2014
        table(border="0" cellspacing="0" cellpadding="0")
            thead
            tr
                th.no #
                th.desc DESCRIPTION
                th.unit UNIT PRICE
                th.qty QUANTITY
                th.total TOTAL
            tbody
            tr
                td.no 01
                td.desc
                h3 Website Design
                | Creating a recognizable design solution based on the company's existing visual identity
                td.unit $40.00
                td.qty 30
                td.total $1,200.00
            tr
                td.no 02
                td.desc
                h3 Website Development
                | Developing a Content Management System-based Website
                td.unit $40.00
                td.qty 80
                td.total $3,200.00
            tr
                td.no 03
                td.desc
                h3 Search Engines Optimization
                | Optimize the site for search engines (SEO)
                td.unit $40.00
                td.qty 20
                td.total $800.00
            tfoot
            tr
                td(colspan="2")
                td(colspan="2") SUBTOTAL
                td $5,200.00
            tr
                td(colspan="2")
                td(colspan="2") TAX 25%
                td $1,300.00
            tr
                td(colspan="2")
                td(colspan="2") GRAND TOTAL
                td $6,500.00
        #thanks Thank you!
        #notices
            div NOTICE:
            .notice A finance charge of 1.5% will be made on unpaid balances after 30 days.
        footer
        | Invoice was created on a computer and is valid without the signature and seal.
```

The line `include style.css` includes our stylesheet `style.css` from the project root.

```

```

    @font-face {
        font-family: SourceSansPro;
        src: url(SourceSansPro-Regular.ttf);
    }

    .clearfix:after {
        content: "";
        display: table;
        clear: both;
    }

    a {
        color: #0087c3;
        text-decoration: none;
    }

    body {
        position: relative;
        width: 21cm;
        height: 29.7cm;
        margin: 0 auto;
        color: #555555;
        background: #ffffff;
        font-family: Arial, sans-serif;
        font-size: 14px;
        font-family: SourceSansPro;
    }

    header {
        padding: 10px 0;
        margin-bottom: 20px;
        border-bottom: 1px solid #aaaaaa;
    }

    #logo {
        float: left;
        margin-top: 8px;
    }

    #logo img {
        height: 70px;
    }

    #company {
        float: right;
        text-align: right;
    }

    #details {
        margin-bottom: 50px;
    }

    #client {
        padding-left: 6px;
        border-left: 6px solid #0087c3;
        float: left;
    }

    #client .to {
        color: #777777;
    }

    h2.name {
        font-size: 1.4em;
        font-weight: normal;
        margin: 0;
    }

    #invoice {
        float: right;
        text-align: right;
    }

    #invoice h1 {
        color: #0087c3;
        font-size: 2.4em;
        line-height: 1em;
        font-weight: normal;
        margin: 0 0 10px 0;
    }

    #invoice .date {
        font-size: 1.1em;
        color: #777777;
    }

    table {
        width: 100%;
        border-collapse: collapse;
        border-spacing: 0;
        margin-bottom: 20px;
    }

    table th,
    table td {
        padding: 20px;
        background: #eeeeee;
        text-align: center;
        border-bottom: 1px solid #ffffff;
    }

    table th {
        white-space: nowrap;
        font-weight: normal;
    }

    table td {
        text-align: right;
    }

    table td h3 {
        color: #57b223;
        font-size: 1.2em;
        font-weight: normal;
        margin: 0 0 0.2em 0;
    }

    table .no {
        color: #ffffff;
        font-size: 1.6em;
        background: #57b223;
    }

    table .desc {
        text-align: left;
    }

    table .unit {
        background: #dddddd;
    }

    table .qty {
    }

    table .total {
        background: #57b223;
        color: #ffffff;
    }

    table td.unit,
    table td.qty,
    table td.total {
        font-size: 1.2em;
    }

    table tbody tr:last-child td {
        border: none;
    }

    table tfoot td {
        padding: 10px 20px;
        background: #ffffff;
        border-bottom: none;
        font-size: 1.2em;
        white-space: nowrap;
        border-top: 1px solid #aaaaaa;
    }

    table tfoot tr:first-child td {
        border-top: none;
    }

    table tfoot tr:last-child td {
        color: #57b223;
        font-size: 1.4em;
        border-top: 1px solid #57b223;
    }

    table tfoot tr td:first-child {
        border: none;
    }

    #thanks {
        font-size: 2em;
        margin-bottom: 50px;
    }

    #notices {
        padding-left: 6px;
        border-left: 6px solid #0087c3;
    }

    #notices .notice {
        font-size: 1.2em;
    }

    footer {
        color: #777777;
        width: 100%;
        height: 30px;
        position: absolute;
        bottom: 0;
        border-top: 1px solid #aaaaaa;
        padding: 8px 0;
        text-align: center;
    }

```


```

Refer the `pug` docs for more info on creating your HTML.

Finally, use `sls deploy` to deploy the lambda function to AWS.
