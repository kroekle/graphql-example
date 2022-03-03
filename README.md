# graphql-example

This example walks you through setting up a local environment to test authorizing GraphQL with Open Policy Agent (OPA)


## Prerequisites 
Ability to run docker compose

DAS Free instance - https://www.styra.com/das-free

Optional: Insomnia or some other tool to execute GraphQL queries


## Initial Setup
1. In your DAS tenant create a new system of type Envoy, give it whatever name you like.  (note the system id, this can be found in the system settings tab)
1. Download the slp config (we will not be using it, but we need some values from it)
    * The first command on the install page will be the slp config, just copy the curl part of the command up to and including the pipe (`|`), then add ` grep -a1 'bearer'`
1. Open the config/opa-conf.yaml and execute the curl command, we will need to replace each of the following in the provided config.
    * Replace `<tenant name>` with the tenant name (1 occurrence), this is in the url of your tenant
    * Replace `<system id>` with the system id (3 occurrences), noted in step 1
    * Replace `<bearer token>` with the bearer token (2 occurrences), from the output of the curl/grep
1. Save the config file and run docker-compose up
    ```sh
    docker-compose up
    ```

You should now have working service that you can hit on 7001

```sh
curl localhost:7001/graphql -H 'Content-Type: application/json' -d '{"query":"{hero {name}}"}'
```

## Rule setup
At this point OPA is making decisions on all the requests, but the default rule says to allow all.  In this section we will setup OPA to be able to write valid rules.  

We must first deal with the fact that a GraphQL document is not JSON, which OPA needs to write rules against.  (future versions of OPA will likely remove the need for this step)

There are several ways we could deal with this, I choose to do a call out to a service that will transform the GraphQL document into AST.  The benefit to this method is that the [reference implementation](https://graphql.org/graphql-js/) of GraphQL has a function to do this, and it was very easy to turn this into a service (code and docker container can be found here: https://github.com/kroekle/gql-to-ast).

I would normally sidecar this service to OPA, but for this example I will just call out to one that I have running in CloudRun `https://gql-to-ast-ndsfor25oq-uc.a.run.app`

The downfall of working with AST is that it can be very complex, I've started to create a rego library to help deal with the complexities (https://github.com/kroekle/gql-rego-library).

1. Copy the contents if the [library](https://github.com/kroekle/gql-rego-library/blob/main/library/graphql/policy.rego) (leaving out the package line) and paste it into our ingress policy in DAS
1. Change the ast_url to `https://gql-to-ast-ndsfor25oq-uc.a.run.app`
1. Replace the existing allow rule with
    ```js
    default allow = false
    default allowed_operation = false
    default restricted_field = false
    default restricted_row = false

    allow {
      allowed_operation
      not restricted_field
      not restricted_row
    }
    ```
1.  Let's add allowed operations for both querying and introspection
    ```js
    allowed_opperation {
      input.parsed_body.operationName == "IntrospectionQuery"
    }

    allowed_opperation {
      ast.definitions[i].kind == "OperationDefinition"
      ast.definitions[i].operation == "query"
    }
    ```
1. Next, let's add a helper function for group membership (I'm cheating here a little by just adding group to a header, normally you would decode a JWT token to get this information)
    ```js
    in_group(groups) {
      input.attributes.request.http.headers.group = groups[_]
    }
    ```

The other thing that this needed to make relevant rules is the GraphQL schema.  Due to the nature of GraphQL, most of the names you deal with in the query document are equivalent to variable names in a programming language.  They are just a reference to something.  By brining the schema into the picture we can turn that something into a type, which is normally much more relevant when creating rules.  Just like the query document, GraphQL schema is not JSON, but we can use the same utility to turn it into AST (which I've already done and placed in this repo)

1. In DAS create a new Data Source and name it schema (don't put it into a package)
   1. use the hamburger menu next to the system and choose `Add Data Source`
   1. Type: `JSON`; Path: `<empty>`;  Data Source Name: `schema`
   1. click save
1. Now edit the Data source and paste the contents of [schema-ast.json](./schema-ast.json)
1. Publish the schema by hitting the Publish button at the top of the page


Now we can add some relevant rules.  First a little explanation of the important rule/functions from the library.

* query_types/mutation_types
   * These rules take the references from the query/mutation document and translate them to their type.  So if you want to make a rule about a property called name on a type called Character you would do something like `query_types["Character"]["name"]`
* query_reference/mutation_reference 
  * These are functions (for now) that work off of the references in the document (normally less useful).  To make a rule about a property called name on a reference name hero you would do something like `query_reference("hero", "name")`
* query_argument/mutation_argument
  * These are functions (for now) that will check equality of an argument.  This is currently working off the references and would be more useful if it was changed to work off of types.  Using it would look something like `query_argument("hero", "id", 2001)`


So here are some potential rules, feel free to play around with them as much as you like
 
```js
restricted_field {
  not in_group(["admin"])
  query_types["Character"]["id"]
}
```

```js
restricted_field {
  not in_group(["admin", "manager"])
  query_reference("friends", "name")
}
```

```js
restricted_row {
  not in_group(["admin"])
  query_argument("hero", "episode", "JEDI")
}
```

## Acknowledgements 
This draws from the great work being done in the following projects
* https://graphql.org/
* https://github.com/open-policy-agent/opa
* https://styra.com

