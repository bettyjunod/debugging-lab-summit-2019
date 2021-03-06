## "Bird Box"... Not Today!!

*15 MINUTES PRACTICE*

The ***Mysterious Application*** in your *Developer Workspaces* is now up and running. It is composed of several components, but so far, you have no clue about how the application is working.
Going all over this application and debugging it completely blindfolded is time consuming and a crazy bid as Malorie does in *Bird Box*.

![BirdBox]({% image_path birdbox.png %}){:width="300px"}

Red Hat OpenShift Container Platform provides services to get observability of applications and to understand how different components are interacting with each other.

#### What is Kiali?

![Kiali]({% image_path kiali.png %}){:width="400px"}

A Microservice Architecture breaks up the monolith into many smaller pieces that are composed together. Patterns to secure the communication between services like fault tolerance (via timeout, retry, circuit breaking, etc.) have come up as well as distributed tracing to be able to see where calls are going.

A service mesh can now provide these services on a platform level and frees the application writers from those tasks. Routing decisions are done at the mesh level.

[Kiali](https://www.kiali.io) works with Istio, in OpenShift or Kubernetes, to visualize the service mesh topology, to provide visibility into features like circuit breakers, request rates and more. It offers insights about the mesh components at different levels, from abstract Applications to Services and Workloads.

#### "Kiali, please tell me, how is the application working?"

Kiali provides an interactive graph view of your namespace in real time, being able to display the interactions at several levels (applications, versions, workloads), with contextual information and charts on the selected graph node or edge.

First, you need to access to Kiali. 
Launch a browser and navigate to [Kiali Console]({{ KIALI_URL }}) *(please make sure to replace **infrax** from the url with your dedicated project)*. 
You should see the Kiali console login screen.

![Kiali - Log In]({% image_path kiali-login.png %}){:width="400px"}

Log in as `{{ OPENSHIFT_USER }}/{{ OPENSHIFT_PASSWORD }}`

After you log in, **click on the Graph link** in the left navigation and enter the following configuration:

 * Namespace: **{{ COOLSTORE_PROJECT }}**
 * Display: **check 'Traffic Animation'**
 * Fetching: **Last min**

![Kiali - Graph]({% image_path kiali-graph.png %}){:width="700px"}

 This page shows a graph with all the microservices, connected by the requests going through them. On this page, you can see how the services interact with each other.

Even if the application *seemed* working fine, there is a problem in the ***Gateway Service*** which sends a 4xx http error.

![Kiali - 4xx]({% image_path kiali-4xx.png %}){:width="300px"}

> In order to get the previous screen, please reload the ***Web UI*** more than one time!

Open the Javascript Console from your browser, and you will find a **404 error** when calling the ***'gateway/api/cart'** API.

![Gateway Error]({% image_path gateway-cart-missing.png %}){:width="700px"}

Indeed, when you check the APIs exposed by the gateway, you cannot find any **'/api/cart/id-*'** one.

Let's fix it!!

#### Build and deploy the Quarkus microservice, the Cart Service

[Quarkus](https://quarkus.io/) is a Kubernetes Native Java stack tailored for GraalVM & OpenJDK HotSpot, crafted from the best of breed Java libraries and standards.

* Architectured for running in serverless and container environments like Knative and OpenShift. 
* Designed around a **container first philosophy**, what this means in real terms is that Quarkus is optimised for low memory usage and fast startup times.

We already compiled the Cart Service application to a native executable called **cart-1.0-SNAPSHOT-runner**. You can find in the **cart-quarkus** project under the **src/target** folder. It improves the startup time of the ***Cart Service***, and produces a minimal disk footprint. The executable would have everything to run the application including the "JVM" and the application.

In this chapter, you will focus on creating a Docker image using the produced native executable.

![Quarkus - Container]({% image_path containerization-process.png %}){:width="700px"}

> If you want, take a moment to examine the source code of the Cart Service implemented with [Quarkus](https://quarkus.io/).
> You can find it under the package ***com.redhat.cloudnative*** in the **src/main/java** directory of the **cart-quarkus** project.

In the ***Terminal Window of CodeReady Workspaces***, execute the following commands to leverage the build mechanism of OpenShift and deploy the service:

~~~shell
# To build the image on OpenShift
$ oc new-build --binary --name=cart -lapp=cart,version=v1.0
$ oc patch bc/cart -p '{"spec":{"strategy":{"dockerStrategy":{"dockerfilePath":"src/main/docker/Dockerfile"}}}}'
$ oc start-build cart --from-dir /projects/labs/cart-quarkus --follow

# To instantiate the image
$ oc new-app --image-stream=cart:latest -lapp=cart,version=v1.0

# To deploy an Istio SideCar and configure Catalog Service Deployment
$ oc rollout pause dc/cart
$ oc patch dc/cart --patch '{"spec": {"template": {"metadata": {"annotations": {"sidecar.istio.io/inject": "true"}}}}}'
$ oc set env dc/cart CATALOG_ENDPOINT=http://catalog:8080
$ oc rollout resume dc/cart

# To create the route
$ oc expose svc cart
~~~

![Openshift Console Cart]({% image_path console-cart.png %}){:width="500px"}

**YOU HAVE TO SEE THAT!** 
Have a look to the log of the ***Cart Service*** pod by cliking in the dark blue circle and then **just admire its amazing FAST BOOT TIME!**

~~~bash
2019-04-01 20:13:35,623 INFO  [io.quarkus] (main) Quarkus 0.11.0 started in 0.009s. Listening on: http://0.0.0.0:8080 
2019-04-01 20:13:35,623 INFO  [io.quarkus] (main) Installed features: [cdi, resteasy, resteasy-jsonb, smallrye-rest-client]
2019-04-01 20:17:08,790 INFO  [com.red.clo.ser.ShoppingCartService] (XNIO-1 task-1) Using local cache for cart data
~~~ 

**AND YES, IT'S A JAVA APPLICATION!**

You can ensure the proper functioning of the ***Cart Service*** by accessing to {{ CART_ROUTE_HOST }} and **click on 'Test it'**.

![Cart Service]({% image_path cart-service.png %}){:width="500px"}

#### Update Gateway Service

Previously, we deployed the ***Cart Service***. Now, you have to take it in account in the ***Gateway Service***.

Under the **gateway-vertx** project, edit the **src/main/java/com/redhat/cloudnative/gateway/GatewayVerticle.java** file as following:

First, uncomment the WebClient attribute ***cart*** in the class ***GatewayVerticle***

~~~java
    private WebClient catalog;
    private WebClient inventory;
    // Cart Attribute
    private WebClient cart; 
~~~

Then, uncomment the **/api/cart/:cardId** route in the ***start()*** method

~~~java
        router.get("/health").handler(ctx -> ctx.response().end(new JsonObject().put("status", "UP").toString()));
        router.get("/api/products").handler(this::products);
        // Cart Route
        router.get("/api/cart/:cardId").handler(this::getCartHandler);
~~~

Next, uncomment the **Cart Lookup** for Service Dicovery in the ***ServiceDiscovery.create()*** call

~~~java
            // Catalog lookup
            Single<WebClient> catalogDiscoveryRequest = HttpEndpoint.rxGetWebClient(discovery,
                    rec -> rec.getName().equals("catalog"))
                    .onErrorReturn(t -> WebClient.create(vertx, new WebClientOptions()
                            .setDefaultHost(System.getProperty("catalog.api.host", "localhost"))
                            .setDefaultPort(Integer.getInteger("catalog.api.port", 9000))));

            // Inventory lookup
            Single<WebClient> inventoryDiscoveryRequest = HttpEndpoint.rxGetWebClient(discovery,
                    rec -> rec.getName().equals("inventory"))
                    .onErrorReturn(t -> WebClient.create(vertx, new WebClientOptions()
                            .setDefaultHost(System.getProperty("inventory.api.host", "localhost"))
                            .setDefaultPort(Integer.getInteger("inventory.api.port", 9001))));

            // Cart lookup
            Single<WebClient> cartDiscoveryRequest = HttpEndpoint.rxGetWebClient(discovery,
                    rec -> rec.getName().equals("cart"))
                    .onErrorReturn(t -> WebClient.create(vertx, new WebClientOptions()
                            .setDefaultHost(System.getProperty("inventory.api.host", "localhost"))
                            .setDefaultPort(Integer.getInteger("inventory.api.port", 9002))));
~~~

Then, replace the ***Single.zip()*** function in the ***ServiceDiscovery.create()*** call as following

~~~java
            // Zip all 3 requests
            Single.zip(catalogDiscoveryRequest, inventoryDiscoveryRequest, cartDiscoveryRequest, 
                (cg, i, ct) -> {
                    // When everything is done
                    catalog = cg;
                    inventory = i;
                    cart = ct;
                    return vertx.createHttpServer()
                        .requestHandler(router::accept)
                        .listen(Integer.getInteger("http.port", 8080));
                }).subscribe();
~~~

Finally, uncomment the ***getCartHandler()*** method in the ***GatewayVerticle*** class.

~~~java
    private void getCartHandler(RoutingContext rc) {
        String cardId = rc.request().getParam("cardId");
        
        // Retrieve cart
        cart
        .get("/api/cart/" + cardId)
        .as(BodyCodec.jsonObject())
        .rxSend()
        .subscribe(
            resp -> {
                if (resp.statusCode() != 200) {
                    new RuntimeException("Invalid response from the cart: " + resp.statusCode());
                }
                rc.response().end(Json.encodePrettily(resp.body()));
            },
            error -> rc.response().end(new JsonObject().put("error", error.getMessage()).toString())
        );
    }
~~~

Check that your source code compiles then use the OpenShift CLI command to start a new build and deployment for the update ***Gateway Service***:

~~~shell
$ mvn clean package -f /projects/labs/gateway-vertx/
$ oc start-build gateway-s2i --from-dir /projects/labs/gateway-vertx/ --follow
~~~

Once deployed, check your javascript console that the **404 error** has disappeared.
In Kiali Graph, the Gateway Service is now green and you can see the new ***Cart Service*** is now present! 

![Gateway Fixed]({% image_path gateway-cart-fixed.png %}){:width="700px"}

**CONGRATULATIONS!!!** You survive and you put off the blindfold on your own. But it is not THE END...

Now,let's go deeper!!