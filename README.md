# Maintenance Mode Proxy

This is an Apigee Edge Proxy that demonstrates how an admin could flip a system into "maintenance mode".


## The idea

In a typical configuration, Apigee Edge acts as a proxy to a backend API implementation. Requests arrive at Edge, Edge performs some validation or check (token, key, header validity, etc), and then if everything is ok, Edge proxies the request to the backend API implementation. 

In some cases, an Operations team wants the ability to easily change a variable in Edge to put the system into "maintennce mode".  In that mode, the API proxy, operating at the edge of the network, does not proxy requests into the configured backend API implementation, but rather responds with a 503 status or similar, stating that the system is in maintenance mode.

The idea is:
- It should be easy to turn the maintenance mode on or off.
- modifications to the maintenance mdoe setting should take effect relatively quickly.
- checks for maintenance mode should not affect performance at scale.
- it should be possible to allow maintenance mode to affect different proxies or products. 

## Key-Value Maps  in Edge

Apigee Edge includes a Key-Value map, which is a persistent store for environmental settings. At runtime, proxies can read from that Key-Value map, and modify behavior based on the values stored there.

The kind of information an ops team could store in the key-value  is wide open. Anything.

## How this proxy works

This proxy works by relying on a Key-Value map with a single key, "maint_mode".
The value is a string.  If the string value is "true", then the proxy responds with a canned 503 response. If the string value is not true, then the proxy responds as normal.

## Testing it

1. import and deploy the API proxy to an Edge environment

2. configure a KeyValue Map called `adminSettings`. (You can use the UI for this)

  a. in the top navbar, hover over APIs, slide down to "environment configuration"

  b. in the resulting page, click the Key Value Maps tab.  Add 'adminSettings'.

  c. add an entry called 'maint_mode' with value 'false'. 

3. invoke the proxy to verify normal behavior.

  ```
  curl -i  -X GET 'https://ORGNAME-ENVNAME.apigee.net/maintenance-mode/t1'
  ```

  You should see something like:

  ```xml
  <Message>
    <system-info>
      <organization.name>ORGNAME</organization.name>
      <environment.name>ENVNAME</environment.name>
      <system.time>Fri, 12 Aug 2016 19:54:25 UTC</system.time>
      <system.timestamp>1471031665146</system.timestamp>
    </system-info>
    <request-info>
      <apiproxy.name>maintenance-mode</apiproxy.name>
      <apiproxy.revision>9</apiproxy.revision>
      <proxy.endpoint>default</proxy.endpoint>
      <client.ip>67.161.89.186</client.ip>
      <messageprocessor_id>15e095d7-1e0e-4779-a256-fb989e9064ee</messageprocessor_id>
      <message_id>rrt-403bc0d4-wo-14719-38927086-1</message_id>
      <request.path>/maintenance-mode/t1</request.path>
      <num_queryparams>
      </num_queryparams>
    </request-info>
  </Message>
  ```
  
4. Use the UI to change maint_mode to "true".

5. invoke the proxy to verify the maintenance mode response.

  ```
  curl -i  -X GET 'https://ORGNAME-ENVNAME.apigee.net/maintenance-mode/t1'
  ```

  You should see something like:

  ```
  <error>
    <code>1540494</code>
    <message>The system is in maintenance mode.</message>
  </error>
  ```

6. use the UI to change maint_mode back to "false"

7. invoke the API again, see the normal response.


## From Curl

You can also turn the maintenance mode ON and OFF via administrative APIs, *if CPS is enabled for the Edge Organization.* "What is CPS?" you may wonder. It's just a different way Apigee manages the stored data for an organization.  For CPS orgs, the KVM Admin API is enhanced.

### Verify CPS

```
curl -i -u username:password -X GET \
 'https://api.enterprise.apigee.com/v1/o/ORGNAME'
 ```

You want to see something like the following in the output of the above command:

```json
 "properties": {
   "property": [
     {
       "value": "true",
       "name": "features.isCpsEnabled"
     }
   ]
 },

```

If you don't see that, then you cannot use the admin calls, to turn on and off maintenance mode.

### Create the KVM

The KVM must be created, before you can set values into it. You can do that like so: 

```
curl -i -H 'content-type: application/json' -X POST \
 -u username:password \
 'https://api.enterprise.apigee.com/v1/o/ORGNAME/e/ENVIRONMENT/keyvaluemaps'\
 -d '{   
 "name" : "adminSettings",
 "entry" : [ { "name" : "maint_mode", "value" : "false" } ]
}'
```


### Turn ON

```
curl -i -u username:password \
 -X POST  -H 'Content-Type: application/json' \
 'https://api.enterprise.apigee.com/v1/o/ORGNAME/e/ENVIRONMENT/keyvaluemaps/adminSettings/entries/maint_mode'\
 -d '{ "value": "true", "name": "maint_mode" }'
```


### Turn OFF

```
curl -i -u username:password \
 -X POST  -H 'Content-Type: application/json' \
 'https://api.enterprise.apigee.com/v1/o/ORGNAME/e/ENVIRONMENT/keyvaluemaps/adminSettings/entries/maint_mode'\
 -d '{ "value": "false", "name": "maint_mode" }'
```


