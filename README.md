# Pact JS workshop

## Introduction

This workshop is aimed at demonstrating core features and benefits of contract testing with Pact.

Whilst contract testing can be applied retrospectively to systems, we will follow the [consumer driven contracts](https://martinfowler.com/articles/consumerDrivenContracts.html) approach in this workshop - where a new consumer and provider are created in parallel to evolve a service over time, especially where there is some uncertainty with what is to be built.

This workshop should take from 1 to 2 hours, depending on how deep you want to go into each topic.

**Workshop outline**:

- [step 1: **create consumer**](https://github.com/pact-foundation/pact-workshop-js/tree/step1#step-1---simple-consumer-calling-provider): Create our consumer before the Provider API even exists
- [step 2: **unit test**](https://github.com/pact-foundation/pact-workshop-js/tree/step2#step-2---client-tested-but-integration-fails): Write a unit test for our consumer
- [step 3: **pact test**](https://github.com/pact-foundation/pact-workshop-js/tree/step3#step-3---pact-to-the-rescue): Write a Pact test for our consumer
- [step 4: **pact verification**](https://github.com/pact-foundation/pact-workshop-js/tree/step4#step-4---verify-the-provider): Verify the consumer pact with the Provider API
- [step 5: **fix consumer**](https://github.com/pact-foundation/pact-workshop-js/tree/step5#step-5---back-to-the-client-we-go): Fix the consumer's bad assumptions about the Provider
- [step 6: **pact test**](https://github.com/pact-foundation/pact-workshop-js/tree/step6#step-6---consumer-updates-contract-for-missing-products): Write a pact test for `404` (missing User) in consumer
- [step 7: **provider states**](https://github.com/pact-foundation/pact-workshop-js/tree/step7#step-7---adding-the-missing-states): Update API to handle `404` case
- [step 8: **pact test**](https://github.com/pact-foundation/pact-workshop-js/tree/step8#step-8---authorization): Write a pact test for the `401` case
- [step 9: **pact test**](https://github.com/pact-foundation/pact-workshop-js/tree/step9#step-9---implement-authorisation-on-the-provider): Update API to handle `401` case
- [step 10: **request filters**](https://github.com/pact-foundation/pact-workshop-js/tree/step10#step-10---request-filters-on-the-provider): Fix the provider to support the `401` case
- [step 11: **pact broker**](https://github.com/pact-foundation/pact-workshop-js/tree/step11#step-11---using-a-pact-broker): Implement a broker workflow for integration with CI/CD
- [step 12: **pactflow broker**](https://github.com/pact-foundation/pact-workshop-js/tree/step12#step-12---using-a-pactflow-broker): Implement a managed pactflow workflow for integration with CI/CD

_NOTE: Each step is tied to, and must be run within, a git branch, allowing you to progress through each stage incrementally. For example, to move to step 2 run the following: `git checkout step2`_

## Learning objectives

If running this as a team workshop format, you may want to take a look through the [learning objectives](./LEARNING.md).

## Requirements

[Docker](https://www.docker.com)

[Docker Compose](https://docs.docker.com/compose/install/)

[Node + NPM](https://nodejs.org/en/)

## Scenario

There are two components in scope for our workshop.

1. Product Catalog website. It provides an interface to query the Product service for product information.
1. Product Service (Provider). Provides useful things about products, such as listing all products and getting the details of an individual product.

## Step 1 - Simple Consumer calling Provider

We need to first create an HTTP client to make the calls to our provider service:

![Simple Consumer](diagrams/workshop_step1.svg)

The Consumer has implemented the product service client which has the following:

- `GET /products` - Retrieve all products
- `GET /products/{id}` - Retrieve a single product by ID

The diagram below highlights the interaction for retrieving a product with ID 10:

![Sequence Diagram](diagrams/workshop_step1_class-sequence-diagram.svg)

You can see the client interface we created in `consumer/src/api.js`:

```javascript
export class API {

    constructor(url) {
        if (url === undefined || url === "") {
            url = process.env.REACT_APP_API_BASE_URL;
        }
        if (url.endsWith("/")) {
            url = url.substr(0, url.length - 1)
        }
        this.url = url
    }

    withPath(path) {
        if (!path.startsWith("/")) {
            path = "/" + path
        }
        return `${this.url}${path}`
    }

    async getAllProducts() {
        return axios.get(this.withPath("/products"))
            .then(r => r.data);
    }

    async getProduct(id) {
        return axios.get(this.withPath("/products/" + id))
            .then(r => r.data);
    }
}
```

After forking or cloning the repository, we may want to install the dependencies `npm install`.
We can run the client with `npm start --prefix consumer` - it should fail with the error below, because the Provider is not running.

![Failed step1 page](diagrams/workshop_step1_failed_page.png)

*Move on to [step 2](https://github.com/pact-foundation/pact-workshop-js/tree/step2#step-2---client-tested-but-integration-fails)*

## Step 2 - Client Tested but integration fails

Now lets create a basic test for our API client. We're going to check 2 things:

1. That our client code hits the expected endpoint
1. That the response is marshalled into an object that is usable, with the correct ID

You can see the client interface test we created in `consumer/src/api.spec.js`:

```javascript
import API from "./api";
import nock from "nock";

describe("API", () => {

    test("get all products", async () => {
        const products = [
            {
                "id": "9",
                "type": "CREDIT_CARD",
                "name": "GEM Visa",
                "version": "v2"
            },
            {
                "id": "10",
                "type": "CREDIT_CARD",
                "name": "28 Degrees",
                "version": "v1"
            }
        ];
        nock(API.url)
            .get('/products')
            .reply(200,
                products,
                {'Access-Control-Allow-Origin': '*'});
        const respProducts = await API.getAllProducts();
        expect(respProducts).toEqual(products);
    });

    test("get product ID 50", async () => {
        const product = {
            "id": "50",
            "type": "CREDIT_CARD",
            "name": "28 Degrees",
            "version": "v1"
        };
        nock(API.url)
            .get('/products/50')
            .reply(200, product, {'Access-Control-Allow-Origin': '*'});
        const respProduct = await API.getProduct("50");
        expect(respProduct).toEqual(product);
    });
});
```



![Unit Test With Mocked Response](diagrams/workshop_step2_unit_test.svg)



Let's run this test and see it all pass:

```console
❯ npm test --prefix consumer

PASS src/api.spec.js
  API
    ✓ get all products (15ms)
    ✓ get product ID 50 (3ms)

Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        1.03s
Ran all test suites.
```

If you encounter failing tests after running `npm test --prefix consumer`, make sure that the current branch is `step2`.

Meanwhile, our provider team has started building out their API in parallel. Let's run our website against our provider (you'll need two terminals to do this):


```console
# Terminal 1
❯ npm start --prefix provider

Provider API listening on port 8080...
```

```console
# Terminal 2
> npm start --prefix consumer

Compiled successfully!

You can now view pact-workshop-js in the browser.

  Local:            http://127.0.0.1:3000/
  On Your Network:  http://192.168.20.17:3000/

Note that the development build is not optimized.
To create a production build, use npm run build.
```

You should now see a screen showing 3 different products. There is a `See more!` button which should display detailed product information.

Let's see what happens!

![Failed page](diagrams/workshop_step2_failed_page.png)

Doh! We are getting 404 everytime we try to view detailed product information. On closer inspection, the provider only knows about `/product/{id}` and `/products`.

We need to have a conversation about what the endpoint should be, but first...

*Move on to [step 3](https://github.com/pact-foundation/pact-workshop-js/tree/step3#step-3---pact-to-the-rescue)*

## Step 3 - Pact to the rescue

Unit tests are written and executed in isolation of any other services. When we write tests for code that talk to other services, they are built on trust that the contracts are upheld. There is no way to validate that the consumer and provider can communicate correctly.

> An integration contract test is a test at the boundary of an external service verifying that it meets the contract expected by a consuming service — [Martin Fowler](https://martinfowler.com/bliki/IntegrationContractTest.html)

Adding contract tests via Pact would have highlighted the `/product/{id}` endpoint was incorrect.

Let us add Pact to the project and write a consumer pact test for the `GET /products/{id}` endpoint.

*Provider states* is an important concept of Pact that we need to introduce. These states help define the state that the provider should be in for specific interactions. For the moment, we will initially be testing the following states:

- `product with ID 10 exists`
- `products exist`

The consumer can define the state of an interaction using the `given` property.

Note how similar it looks to our unit test:

In `consumer/src/api.pact.spec.js`:

```javascript
import path from "path";
import { PactV3, MatchersV3, SpecificationVersion, } from "@pact-foundation/pact";
import { API } from "./api";
const { eachLike, like } = MatchersV3;

const provider = new PactV3({
  consumer: "FrontendWebsite",
  provider: "ProductService",
  log: path.resolve(process.cwd(), "logs", "pact.log"),
  logLevel: "warn",
  dir: path.resolve(process.cwd(), "pacts"),
  spec: SpecificationVersion.SPECIFICATION_VERSION_V2,
  host: "127.0.0.1"
});

describe("API Pact test", () => {
  describe("getting all products", () => {
    test("products exists", async () => {
      // set up Pact interactions
      await provider.addInteraction({
        states: [{ description: "products exist" }],
        uponReceiving: "get all products",
        withRequest: {
          method: "GET",
          path: "/products",
        },
        willRespondWith: {
          status: 200,
          headers: {
            "Content-Type": "application/json; charset=utf-8",
          },
          body: eachLike({
            id: "09",
            type: "CREDIT_CARD",
            name: "Gem Visa",
          }),
        },
      });

      await provider.executeTest(async (mockService) => {
        const api = new API(mockService.url);

        // make request to Pact mock server
        const product = await api.getAllProducts();

        expect(product).toStrictEqual([
          { id: "09", name: "Gem Visa", type: "CREDIT_CARD" },
        ]);
      });
    });
  });

  describe("getting one product", () => {
    test("ID 10 exists", async () => {
      // set up Pact interactions
      await provider.addInteraction({
        states: [{ description: "product with ID 10 exists" }],
        uponReceiving: "get product with ID 10",
        withRequest: {
          method: "GET",
          path: "/products/10",
        },
        willRespondWith: {
          status: 200,
          headers: {
            "Content-Type": "application/json; charset=utf-8",
          },
          body: like({
            id: "10",
            type: "CREDIT_CARD",
            name: "28 Degrees",
          }),
        },
      });

      await provider.executeTest(async (mockService) => {
        const api = new API(mockService.url);

        // make request to Pact mock server
        const product = await api.getProduct("10");

        expect(product).toStrictEqual({
          id: "10",
          type: "CREDIT_CARD",
          name: "28 Degrees",
        });
      });
    });
  });
});
```


![Test using Pact](diagrams/workshop_step3_pact.svg)

This test starts a mock server a random port that acts as our provider service. To get this to work we update the URL in the `Client` that we create, after initialising Pact.

To simplify running the tests, add this to `consumer/package.json`:

```javascript
// add it under scripts
"test:pact": "cross-env CI=true react-scripts test --testTimeout 30000 pact.spec.js",
```

Running this test still passes, but it creates a pact file which we can use to validate our assumptions on the provider side, and have conversation around.

```console
❯ npm run test:pact --prefix consumer

PASS src/api.spec.js
PASS src/api.pact.spec.js

Test Suites: 2 passed, 2 total
Tests:       4 passed, 4 total
Snapshots:   0 total
Time:        2.792s, estimated 3s
Ran all test suites.
```

A pact file should have been generated in *consumer/pacts/FrontendWebsite-ProductService.json*

*NOTE*: even if the API client had been graciously provided for us by our Provider Team, it doesn't mean that we shouldn't write contract tests - because the version of the client we have may not always be in sync with the deployed API - and also because we will write tests on the output appropriate to our specific needs.

*Move on to [step 4](https://github.com/pact-foundation/pact-workshop-js/tree/step4#step-4---verify-the-provider)*

## Step 4 - Verify the provider

We need to make the pact file (the contract) that was produced from the consumer test available to the Provider module. This will help us verify that the provider can meet the requirements as set out in the contract. For now, we'll hard code the path to where it is saved in the consumer test, in step 11 we investigate a better way of doing this.

Now let's make a start on writing Pact tests to validate the consumer contract:

In `provider/product/product.pact.test.js`:

```javascript
const { Verifier } = require('@pact-foundation/pact');
const path = require('path');

// Setup provider server to verify
const app = require('express')();
app.use(require('./product.routes'));
const server = app.listen("8080");

describe("Pact Verification", () => {
    it("validates the expectations of ProductService", () => {
        const opts = {
            logLevel: "INFO",
            providerBaseUrl: "http://127.0.0.1:8080",
            provider: "ProductService",
            providerVersion: "1.0.0",
            pactUrls: [
                path.resolve(__dirname, '../../consumer/pacts/FrontendWebsite-ProductService.json')
            ]
        };

        return new Verifier(opts).verifyProvider().then(output => {
            console.log(output);
        }).finally(() => {
            server.close();
        });
    })
});
```

To simplify running the tests, add this to `provider/package.json`:

```javascript
// add it under scripts
"test:pact": "npx jest --testTimeout=30000 --testMatch \"**/*.pact.test.js\""
```

We now need to validate the pact generated by the consumer is valid, by executing it against the running service provider, which should fail:

```console
❯ npm run test:pact --prefix provider

Verifying a pact between FrontendWebsite and ProductService

  get product with ID 10
    returns a response which
      has status code 200 (FAILED)
      includes headers
        "Content-Type" with value "application/json; charset=utf-8" (FAILED)
      has a matching body (FAILED)

  get all products
    returns a response which
      has status code 200 (OK)
      includes headers
        "Content-Type" with value "application/json; charset=utf-8" (OK)
      has a matching body (OK)


Failures:

1) Verifying a pact between FrontendWebsite and ProductService Given product with ID 10 exists - get product with ID 10
    1.1) has a matching body
           expected 'application/json;charset=utf-8' body but was 'text/html;charset=utf-8'
    1.2) has status code 200
           expected 200 but was 404
    1.3) includes header 'Content-Type' with value 'application/json; charset=utf-8'
           Expected header 'Content-Type' to have value 'application/json; charset=utf-8' but was 'text/html; charset=utf-8'
```

![Pact Verification](diagrams/workshop_step4_pact.svg)

The test has failed, as the expected path `/products/{id}` is returning 404. We incorrectly believed our provider was following a RESTful design, but the authors were too lazy to implement a better routing solution 🤷🏻‍♂️.

The correct endpoint which the consumer should call is `/product/{id}`.

Move on to [step 5](https://github.com/pact-foundation/pact-workshop-js/tree/step5#step-5---back-to-the-client-we-go)
