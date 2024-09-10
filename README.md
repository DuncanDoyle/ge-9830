# Gloo-9830 Reproducer

Issue: https://github.com/solo-io/gloo/issues/9830

## Installation

Add Gloo EE Helm repo:
```
helm repo add glooe https://storage.googleapis.com/gloo-ee-helm
```

Export your Gloo Edge License Key to an environment variable:
```
export GLOO_EDGE_LICENSE_KEY={your license key}
```

Install Gloo Edge:
```
cd install
./install-gloo-edge-enterprise-with-helm.sh
```

> NOTE
> The Gloo Edge version that will be installed is set in a variable at the top of the `install/install-gloo-edge-enterprise-with-helm.sh` installation script.

## Setup the environment

Run the `install/setup.sh` script to setup the environment:

- Deploy the HTTPBin application
- Deploy the RouteTables
- Deploy the VirtualServices


```
./setup.sh
```

## Call Petstore

Call the httpbin app using the following URL and headers which will match the first matcher in `routetables/httpbin-rt.yaml`

```
curl -v -H "city-id: 99999" http://api.example.com/get
```

You will get a response from the `httpbin-httpbin-8000` upstream, as it's defined before the direct response action in the `httpbin` routetable.

Now, deploy the second routetable, `routetables/httpbin2-rt.yaml`, which will also be selected by the `api-example-com` VirtualService:

```
kubectl apply -f routetables/httpbin2-rt.yaml
```

Note that this routetables does not contain any matchers that would match the request we've sent earlier. Now, send the same http request again:

```
curl -v -H "city-id: 99999" http://api.example.com/get
```

... and notice that instead of getting the same response from the `httpbin-httpbin-8000` Upstream, we now receive the direct response defined in the `httpbin-rt` routetable (the original routetable, not the last one we've deployed).

This implies that the ordering of the matchers has changed, and Gloo no longer adheres the order of the routes in the routetable.

This can also be observed from the Envoy config dumps in the `configdumps/` directory:
- `envoy-config-dump-one-rt.yaml`: Config dump when only the single routetable is selected by the VirtualService.
- `envoy-config-dump-two-rt.yaml`: Config dump when both routetables are selected by the VirtualService

Observe that in the second config dump, the order of the routes defined in the `httpbin-rt.yaml` has changed.

One RouteTable:

```
"dynamic_route_configs": [
  {
    "version_info": "721925422575290896",
    "route_config": {
    "@type": "type.googleapis.com/envoy.config.route.v3.RouteConfiguration",
    "name": "listener-::-8080-routes",
    "virtual_hosts": [
      {
      "name": "gloo-system_api-example-com",
      "domains": [
        "api.example.com"
      ],
      "routes": [
        {
        "match": {
          "prefix": "/",
          "headers": [
          {
            "name": "city-id",
            "exact_match": "99999"
          }
          ]
        },
        "route": {
          "cluster": "httpbin-httpbin-8000_gloo-system"
        },
        "name": "gloo-system_api-example-com-route-0-matcher-0"
        },
        {
        "match": {
          "prefix": "/get"
        },
        "direct_response": {
          "status": 200,
          "body": {
          "inline_string": "DIRECT1"
          }
        },
        "name": "gloo-system_api-example-com-route-1-matcher-0"
        }
      ],
      "typed_per_filter_config": {
        "envoy.filters.http.ext_authz": {
        "@type": "type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthzPerRoute",
        "disabled": true
        }
      }
      }
    ]
    },
    "last_updated": "2024-09-10T13:45:20.260Z"
  }
]
```

Two RouteTables:
```
"dynamic_route_configs": [
  {
    "version_info": "14667859535387287171",
    "route_config": {
    "@type": "type.googleapis.com/envoy.config.route.v3.RouteConfiguration",
    "name": "listener-::-8080-routes",
    "virtual_hosts": [
      {
      "name": "gloo-system_api-example-com",
      "domains": [
        "api.example.com"
      ],
      "routes": [
        {
        "match": {
          "prefix": "/get"
        },
        "direct_response": {
          "status": 200,
          "body": {
          "inline_string": "DIRECT1"
          }
        },
        "name": "gloo-system_api-example-com-route-0-matcher-0"
        },
        {
        "match": {
          "prefix": "/anything/test",
          "headers": [
          {
            "name": "city-id",
            "exact_match": "10000"
          }
          ]
        },
        "route": {
          "cluster": "httpbin-httpbin-8000_gloo-system"
        },
        "name": "gloo-system_api-example-com-route-1-matcher-0"
        },
        {
        "match": {
          "prefix": "/anything/rider"
        },
        "direct_response": {
          "status": 200,
          "body": {
          "inline_string": "DIRECT2"
          }
        },
        "name": "gloo-system_api-example-com-route-2-matcher-0"
        },
        {
        "match": {
          "prefix": "/",
          "headers": [
          {
            "name": "city-id",
            "exact_match": "99999"
          }
          ]
        },
        "route": {
          "cluster": "httpbin-httpbin-8000_gloo-system"
        },
        "name": "gloo-system_api-example-com-route-3-matcher-0"
        }
      ],
      "typed_per_filter_config": {
        "envoy.filters.http.ext_authz": {
        "@type": "type.googleapis.com/envoy.extensions.filters.http.ext_authz.v3.ExtAuthzPerRoute",
        "disabled": true
        }
      }
      }
    ]
    },
    "last_updated": "2024-09-10T13:50:01.266Z"
  }
]
```

## Conclusion
TODO