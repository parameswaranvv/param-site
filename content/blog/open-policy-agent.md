---
title: "API Authorization using Open Policy Agent (OPA) as a Sidecar Container"
date: 2020-04-13T15:59:28-05:00
draft: false
description: "Using Open Policy Agent as a Sidecar container for REST Endpoint Authorization"
tags: 
    - Open Policy Agent
categories:
    - Security policy as code
    - Policy Engine
author: Param V V
ReadingTime: 20 min
---

## Problem Statement

In today's cloud native world, authorization and policies are more complex than ever. In a typical enterprise that has been using RBAC (Role Based 
Access Control) traditionally, the policy management becomes a nightmare because the need for creation of new users, roles, controls keeps growing 
based on everyday's needs. Besides, enforcing these mammoth set of rules or policies to secure huge monolithic application makes it even more harder to 
manage and maintain. 

There are organizations where policies are managed using Excel files or Word/PDF documents, and then implemented at the ground level. This leads to 
opportunities to make errors, making testing the policy changes really difficult or not possible, and rolling out new changes is extremely cumbersome.

OPA is a practical solution to solve these problems, especially for a cloud native world where the industry is moving to.

## Overview of OPA

### Key Features

* __`Open Source` (backed by CNCF):__ 
    <p> OPA is an `open source`, `general-purpose policy engine (of the CNCF)` that unifies policy enforcement across the stack. OPA provides a high-level declarative language that let’s you specify policy as code and simple APIs to offload policy decision-making from your software. You can use OPA to enforce policies in microservices, Kubernetes admission control, CI/CD pipelines, API gateways, and more. </p>

* __Policy Engine with `Policies as first-class citizens`:__ 
    <p> OPA treats `Policies as First Class Citizens`. Policies have their own lifecycle and toolset in OPA, which means you can manage policies on their own. This allows decoupling the applications/infrastructure using the policies from the policy management itself. </p>

* __Policies can be `Test-Driven`:__
    <p>OPA supports testing policies in isolation, which makes it more robust and scalable. In a distributed architecture world, this allows to gain more confidence by integrating the testing, and deployment process via a CI/CD pipeline.</p>
  
* __OPA is `context-aware` and decides based on the current context of the input, data and policy rules.__  
    It means that the policy decision making uses the current context information to make the decisions. An Administrator can 
modify policies based on real world current situations, and OPA would take that into consideration right away.

At the outset, there are two aspects to security policy enforcement. 

* `Policy Enforcement` - This is the responsibility of the application to enforce the decisions of the policy evaluation outcomes.
* `Policy Decision Making` - This is the responsibility of the OPA Engine based on the data, input and policies available.

OPA simplies this by decoupling the maintaince of policies, providing access to decision making endpoints using structured data, and enforcement. 
When your software needs to make policy decisions it queries OPA and supplies structured data (e.g., JSON) as input. 
OPA accepts arbitrary structured data as input.

### How does this help Developer Productivity?

Developers need not worry about designing the security access controls beforehand. Since the policies and decision making is decoupled, and uses the current context in time, 
OPA allows the developers to focus on building the software, and parallely or later on makes changes to the authorization policies, without affecting the application.

Since OPA policies can be test-driven, it makes it easier to decouple the application development from the Security policies development.

<div style="alignment: center"><img src="/images/OPA/OPA-Overview.png" /></div></br>

### How is a decision made by the OPA Engine?

The OPA decision making process involves three pieces of information namely:

* `Data` - The supporting data(in JSON) that OPA would need to make a decision. Eg: a list of ACLs with Users, Roles and Permissions to determine the access decision of a current user.
* `Input` - The request(in JSON) to OPA to determine a decision. Eg: Is a user Ken allowed access to a resource?
* `Policy` - A policy statement which specifies the logic to compute the decision using the input and data. Written in a specific language called `Rego`

## Practical Example - Secure REST Endpoints using OPA

The codebase for the below example is available [here](https://github.com/parameswaranvv/opa-demo.git)

The steps below outline the implementation guidelines for a sample architecture, wherein we will secure two REST Endpoints and implement 
Authorization using OPA deployed as a Sidecar in a Kubernetes Pod. 

For simplicity, the deployment architecture shows a simple Master connected with 2 worker nodes to form a cluster.

<div style="alignment: center"><img src="/images/OPA/OPA_Architecture.png" /></div></br>

### Web application setup

I used Spring Boot with Spring Security to expose two REST Endpoints `/hello` and `/bye`. Both the endpoints are protected by Basic Authentication using an in-memory authentication mechanism.
__Note:__ Authentication is not really the focus of this example. The goal here is to demonstrate the authorization decision making using the login roles.

Assuming that you are familiar with basic Spring Boot setup, if not please use the [Spring Initializer](https://start.spring.io/) to bootstrap a basic Spring Boot project with Web dependency.

The two REST endpoints are simply as follows:

```java
		@GetMapping("/hello")
		public String sayHello() {
			return "Hello World";
		}

		@GetMapping("/bye")
		public String sayBye() {
			return "Bye";
		}
```

For a basic Spring Security setup, please refer to the sample code below. Please do not hardcode credentials in a real environment.

__WebSecurityConfiguration.java__
```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfiguration extends WebSecurityConfigurerAdapter {

	@Value("${opa.auth.url}")
	private String opaAuthURL;

	@Autowired
	private AuthenticationEntryPoint authEntryPoint;

	@Autowired
	public void configureGlobal(AuthenticationManagerBuilder auth) throws Exception {
		auth.inMemoryAuthentication()
				.withUser("john123").password(passwordEncoder().encode("password")).roles("USER")
				.and()
				.withUser("admin123").password(passwordEncoder().encode("admin")).roles("ADMIN");
	}

	@Bean
	public PasswordEncoder passwordEncoder() {
		return new BCryptPasswordEncoder();
	}

	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.csrf().disable().authorizeRequests()
				.accessDecisionManager(accessDecisionManager())
				.anyRequest().authenticated().and().httpBasic();
	}

	@Bean
	public AccessDecisionManager accessDecisionManager() {
		List<AccessDecisionVoter<? extends Object>> decisionVoters = Arrays
				.asList(new OPAVoter(opaAuthURL));
		return new UnanimousBased(decisionVoters);
	}

}
```
__AuthenticationEntryPoint.java__
```java
@Component
public class AuthenticationEntryPoint extends BasicAuthenticationEntryPoint {

	@Override
	public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authEx)
			throws IOException {
		response.addHeader("WWW-Authenticate", "Basic realm=" +getRealmName());
		response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
		PrintWriter writer = response.getWriter();
		writer.println("HTTP Status 401 - " + authEx.getMessage());
	}

	@Override
	public void afterPropertiesSet() {
		setRealmName("DeveloperStack");
		super.afterPropertiesSet();
	}

}
```

__Note:__ The important thing to note here is that OPA exposes REST endpoints to allow invocations for decision making. To instruct Spring Security to interact with OPA, we create a 
bean for `AccessDecisionManager` and register it. The `AccessDecisionManager` is a custom code that takes care of interaction with OPA. Sample code is below.

__OPAVoter.java__
```java

public class OPAVoter implements AccessDecisionVoter<Object> {

	private String opaAuthURL;

	private RestTemplate restTemplate;

	public OPAVoter(String opaAuthURL) {
		this.opaAuthURL = opaAuthURL;
		this.restTemplate = new RestTemplate();
	}

	@Override
	public boolean supports(ConfigAttribute attribute) {
		return true;
	}

	@Override
	public boolean supports(Class<?> clazz) {
		return true;
	}

	@Override
	public int vote(Authentication authentication, Object obj, Collection<ConfigAttribute> attributes) {

		if (!(obj instanceof FilterInvocation)) {
			return ACCESS_ABSTAIN;
		}

		FilterInvocation filter = (FilterInvocation) obj;
		Map<String, String> headers = new HashMap<String, String>();

		for (Enumeration<String> headerNames = filter.getRequest().getHeaderNames(); headerNames.hasMoreElements();) {
			String header = headerNames.nextElement();
			headers.put(header, filter.getRequest().getHeader(header));
		}

		String[] path = filter.getRequest().getRequestURI().replaceAll("^/|/$", "").split("/");

		Map<String, Object> input = new HashMap<String, Object>();
		input.put("roles", authentication.getAuthorities());
		input.put("method", filter.getRequest().getMethod());
		input.put("path", path);

		HttpEntity<?> request = new HttpEntity<>(new OPADataRequest(input));
		OPADataResponse response = restTemplate.postForObject(this.opaAuthURL, request, OPADataResponse.class);

		if (!response.getResult()) {
			return ACCESS_DENIED;
		}

		return ACCESS_GRANTED;
	}

}

```

__OPADataRequest.java__
```java
public class OPADataRequest {

	Map<String, Object> input;

	public OPADataRequest(Map<String, Object> input) {
		this.input = input;
	}

	public Map<String, Object> getInput() {
		return this.input;
	}

}
```

__OPADataResponse.java__
```java
public class OPADataResponse {

	private boolean result;

	public OPADataResponse() {
	}

	public boolean getResult() {
		return this.result;
	}

	public void setResult(boolean result) {
		this.result = result;
	}

}
```

I have a pipeline that builds the application, creates the Docker image and pushes to the Docker Registry as `parameswaranvv/opa-web-demo:latest`. 
__Dockerfile__
```dockerfile
FROM openjdk:8-jdk-alpine
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
ARG DEPENDENCY=build/dependency
COPY ${DEPENDENCY}/BOOT-INF/lib /app/lib
COPY ${DEPENDENCY}/META-INF /app/META-INF
COPY ${DEPENDENCY}/BOOT-INF/classes /app
ENTRYPOINT ["java","-cp","app:app/lib/*","com.example.opawebdemo.OpaWebDemoApplication"]
```
### Defining OPA Policies

OPA uses `Rego` for defining Policies. For this example, since we have two endpoints, we will implement rules to allow access as follows:

* `/hello` endpoint is allowed for logged in users with `ROLE_USER` role.
```text
allow {
  input.method == "GET"
  input.path = ["hello"]
  is_user
}
# user is allowed if he has a user role
is_user {

	# for some `i`...
	some i

  input.roles[i].authority == "ROLE_USER"
}
```

* `/bye` endpoint is allowed for logged in users with `ROLE_ADMIN` role.

```text
allow {
  input.method == "GET"
  input.path = ["bye"]
  is_admin
}
# user is allowed if he has a admin role
is_admin {

	# for some `i`...
	some i

  input.roles[i].authority == "ROLE_ADMIN"
}
```

Putting this together, we have

__opaweb-policy.rego__
```text
package opaweb.authz

default allow = false

allow {
  input.method == "GET"
  input.path = ["hello"]
  is_user
}

allow {
  input.method == "GET"
  input.path = ["bye"]
  is_admin
}


# user is allowed if he has a user role
is_user {

	# for some `i`...
	some i

  input.roles[i].authority == "ROLE_USER"
}

# user is allowed if he has a admin role
is_admin {

	# for some `i`...
	some i

  input.roles[i].authority == "ROLE_ADMIN"
}
```

### Testing our defined policies

OPA allows us to test the policies in isolation, without the real need for an application. This allows us to scale and deploy policies as needed in a distributed fashion. This 
is also a fantastic feature of OPA that allows us to deliver Policy changes in a CI/CD manner.

__opaweb-policy-test.rego__
```text
package opaweb.authz

test_hello_allowed_for_user {
    allow with input as {"path": ["hello"], "method": "GET", "roles": [{"authority": "ROLE_USER"}]}
}

test_hello_denied_for_others {
    not allow with input as {"path": ["hello"], "method": "GET", "roles": [{"authority": "ROLE_ADMIN"}]}
    not allow with input as {"path": ["hello"], "method": "GET", "roles": [{"authority": "ROLE_ANONYMOUS"}]}
}

test_bye_allowed_for_admin {
    allow with input as {"path": ["bye"], "method": "GET", "roles": [{"authority": "ROLE_ADMIN"}]}
}

test_bye_denied_for_others {
    not allow with input as {"path": ["bye"], "method": "GET", "roles": [{"authority": "ROLE_USER"}]}
    not allow with input as {"path": ["bye"], "method": "GET", "roles": [{"authority": "ROLE_ANONYMOUS"}]}
}
```

To run the tests, just do

```shell script
$ ./opa test . --verbose
data.opaweb.authz.test_hello_allowed_for_user: PASS (623.171µs)
data.opaweb.authz.test_hello_denied_for_others: PASS (212.274µs)
data.opaweb.authz.test_bye_allowed_for_admin: PASS (184.635µs)
data.opaweb.authz.test_bye_denied_for_others: PASS (274.028µs)
--------------------------------------------------------------------------------
PASS: 4/4
```

### Run locally and check

* Instructions to run OPA locally
    * Refer [here](https://www.openpolicyagent.org/docs/latest/#running-opa) for instructions on downloading OPA to your machine.
    * Alternative, use Docker to spin it up instantly.
    ```shell script
    $ docker run -p 8181:8181 openpolicyagent/opa run --server --log-level debug
    ```

* Register the Policy definition in the OPA server using the available REST endpoint.
    ```shell script
    $ curl --location --request PUT 'localhost:8181/v1/policies/opaweb/authz' \
      --header 'Content-Type: text/plain' \
      --data-binary @opaweb-policy.rego 
    ``` 
  
* Run the Spring Boot application. Assuming the app is exposed on port `8080`, you can try invoking the following endpoints to received a HTTP OK response with the appropriate response body.:
    ```shell script
    $ curl -kv http://localhost:8080/hello --header 'Authorization: Basic am9objEyMzpwYXNzd29yZA=='
    ```
    ```shell script
    $ curl -kv http://localhost:8080/bye --header 'Authorization: Basic YWRtaW4xMjM6YWRtaW4='
    ```
### Deploying in a Kubernetes Environment

For the above example, I created a Deployment definition yaml that I used to deploy in my Kubernetes cluster. The deployment specifies a Pod containing two containers:
* the webapp
* OPA server

But, how do we provide the policies to the server? Calling REST endpoints at startup, in a Production grade environment is not really recommended. Instead, we can create ConfigMaps or Secrets
with our Policy files, and mount them as volumes in the OPA Container.

__configmap.yml__
```shell script
$ kubectl create configmap opademo-policy --from-file=opaweb-policy.rego
```

__deployment.yml__
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opademo
  labels:
    app: opademo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: opademo
  template:
    metadata:
      labels:
        app: opademo
      name: opademo
    spec:
      containers:
        - name: webapp
          image: parameswaranvv/opa-web-demo:latest
          ports:
            - name: http
              containerPort: 8080
          env:
            - name: opa.auth.url
              value: http://localhost:8181/v1/data/opaweb/authz/allow
        - name: opa
          image: openpolicyagent/opa:latest
          ports:
            - name: http
              containerPort: 8181
          args:
            - "run"
            - "--ignore=.*"  # exclude hidden dirs created by Kubernetes
            - "--server"
            - "/policies"
          volumeMounts:
            - readOnly: true
              mountPath: /policies
              name: opademo-policy
          livenessProbe:
            httpGet:
              scheme: HTTP              # assumes OPA listens on localhost:8181
              port: 8181
            initialDelaySeconds: 5      # tune these periods for your environemnt
            periodSeconds: 5
          readinessProbe:
              httpGet:
                path: /health?bundle=true  # Include bundle activation in readiness
                scheme: HTTP
                port: 8181
              initialDelaySeconds: 5
              periodSeconds: 5
      volumes:
        - name: opademo-policy
          configMap:
            name: opademo-policy
```

We then go on to expose a Service for our webapp, so that we can access it via the Service.

__service.yml__
```yaml
apiVersion: v1
kind: Service
metadata:
  name: opa-demo
spec:
  selector:
    app: opademo
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

Done!! You can test the webapp now using your Service IP address and reuse the same CURL commands from above.

## References
* [Example codebase](https://github.com/parameswaranvv/opa-demo.git)
* [OPA Documentation](https://www.openpolicyagent.org/docs/latest/)
* [Why was OPA necessary and case studies?](https://opensource.com/article/19/8/open-policy-agent)
