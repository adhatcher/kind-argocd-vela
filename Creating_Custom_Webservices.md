# Examples of how to customize Webservice experience in Kubevela

## Two clean ways to “pre-wire” a webservice experience in KubeVela:

1. Wrap/clone webservice into your own ComponentDefinition (best when you want defaults like probes, ports, resources baked in).
2. Keep webservice as-is, but create opinionated TraitDefinitions (best when you want “just add trait X” with sane defaults).

In practice, most platforms do both: component defaults for “day-1”, traits for “day-2”.

### 1. Create an opinionated web component type (defaults for probes, ports, resources)
As a platform engineer, you define a new component type (example: standard-webservice) and set default parameters for readiness/liveness probes (and anything else you want standardized).
KubeVela docs show how to build custom components via ComponentDefinition.
Example ComponentDefinition (CUE template) with probe defaults (users can still override):

```YAML
apiVersion: core.oam.dev/v1beta1
kind: ComponentDefinition
metadata:
  name: standard-webservice
  namespace: vela-system
  annotations:
    definition.oam.dev/description: "Opinionated webservice with default probes/resources"
spec:
  workload:
    definition:
      apiVersion: apps/v1
      kind: Deployment
  schematic:
    cue:
      template: |
        parameter: {
          image: string
          port: *8080 | int

          # Defaults; app teams can override
          probes: {
            readiness: *{
              httpGet: { path: "/ready", port: parameter.port }
              initialDelaySeconds: *5 | int
              periodSeconds: *10 | int
              failureThreshold: *3 | int
            } | {...}

            liveness: *{
              httpGet: { path: "/healthz", port: parameter.port }
              initialDelaySeconds: *10 | int
              periodSeconds: *10 | int
              failureThreshold: *3 | int
            } | {...}
          }

          resources: *{
            requests: { cpu: "10m", memory: "80Mi" }
            limits:   { memory: "128Mi" }
          } | {...}
        }

        output: {
          apiVersion: "apps/v1"
          kind: "Deployment"
          metadata: name: context.name
          spec: {
            selector: matchLabels: { "app.oam.dev/component": context.name }
            template: {
              metadata: labels: { "app.oam.dev/component": context.name }
              spec: containers: [{
                name: context.name
                image: parameter.image
                ports: [{ containerPort: parameter.port }]
                resources: parameter.resources
                readinessProbe: parameter.probes.readiness
                livenessProbe:  parameter.probes.liveness
              }]
            }
          }
        }

        # Optional: also generate a Service so http-route can target it
        outputs: service: {
          apiVersion: "v1"
          kind: "Service"
          metadata: name: context.name
          spec: {
            selector: { "app.oam.dev/component": context.name }
            ports: [{
              port: parameter.port
              targetPort: parameter.port
              name: "http"
            }]
          }
        }
```


### 2. Attach “standard” traits for topology spread, HTTPRoute, and HPA

#### Topology spread constraints

KubeVela has a built-in topologyspreadconstraints trait type (it patches workload pod templates under spec.template). [Docs](https://kubevela.io/docs/end-user/traits/references/?utm_source=chatgpt.com)

##### HTTPRoute

KubeVela supports http-route as a trait (commonly provided by an addon like Traefik/Gateway). The docs show http-route trait usage with domains, rules, gatewayName, listenerName. [Docs](https://kubevela.io/docs/tutorials/access-application/?utm_source=chatgpt.com)
(Heads-up: this trait may not exist until the relevant addon/capability is installed—people hit “trait definition … not found” when it’s missing.[Docs](https://github.com/kubevela/kubevela/issues/7013?utm_source=chatgpt.com) )

##### HPA

There is a built-in hpa TraitDefinition example in the KubeVela definition protocol docs.[Docs](https://kubevela.io/docs/platform-engineers/oam/x-definition/?utm_source=chatgpt.com)
Also: if you use HPA (or KEDA), you typically want an “apply-once” style policy so the controller owns replicas after the first apply (KubeVela docs call this out for autoscaling). [Docs](https://kubevela.io/docs/tutorials/auto-scaler/?utm_source=chatgpt.com)

### Putting it together: an Application that “just works” with your defaults + traits

```YAML
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: demo
spec:
  components:
    - name: frontend
      type: standard-webservice
      properties:
        image: ghcr.io/acme/frontend:2026.02.14-1
        # port/probes/resources can be omitted (defaults) or overridden

      traits:
        - type: topologyspreadconstraints
          properties:
            constraints:
              - maxSkew: 1
                topologyKey: kubernetes.io/hostname
                whenUnsatisfiable: DoNotSchedule
                labelSelector:
                  matchLabels:
                    app.oam.dev/component: frontend

        - type: http-route
          properties:
            domains: ["frontend.example.com"]
            gatewayName: traefik-gateway
            listenerName: web
            rules:
              - port: 8080
                path: /

        - type: hpa
          properties:
            min: 2
            max: 10
            cpuPercent: 70

  # Optional but recommended with autoscaling so replicas aren't constantly reconciled by the app
  policies:
    - name: apply-once
      type: apply-once
```

If you want “no-config traits” (predefined defaults when users don’t set properties)

Do this by creating your own TraitDefinitions (wrappers) like:
standard-topologyspread (no properties; always applies your org constraints)
standard-public-route (maybe only needs domain, everything else defaulted)
standard-hpa (defaults min/max/metrics)
TraitDefinition customization is explicitly supported (traits can also patch generated workload/service output).

If you tell me which gateway controller/addon you’re using for HTTPRoute (Traefik? Envoy Gateway? something else) and what “standard” probe paths/thresholds you want, I’ll tailor the exact TraitDefinition CUE for standard-public-route and standard-hpa so app teams can literally add - type: standard-hpa with zero properties.

## If your goal is “app teams write the fewest possible lines,” the winning pattern in KubeVela is

- **One opinionated component type** (your “golden webservice”) with as many defaults as you can safely bake in (ports, probes, resources, envFrom, service creation, etc.)
- **A small set of “opinionated wrapper traits”** that apply defaults automatically (topology spread, http exposure, autoscaling)
- Optionally, **a workflow step** or policy that auto-attaches those traits so teams don’t even list them

### Make a “golden webservice” component type

Instead of using the stock `webservice`, create `webservice.default` (or similar) as a `ComponentDefinition` that renders:

- Deployment
- Service
- default readiness/liveness probes
- default resources
- default labels/selectors
- default configMap envFrom (if you always use it)

Then apps only specify `image` (and maybe domain).

Why: probes are not “optional add-ons” for most orgs—they’re baseline hygiene. Component defaults are the cleanest place to enforce them.

**Key trick**: allow overrides, but provide strong defaults via *{...} | {...} patterns in CUE.

### 2. Create “no-config” wrapper traits for everything else

Even if KubeVela already has traits like `hpa`, `topologyspreadconstraints`, `http-route`, you can wrap them with your own traits that have:

- zero required fields
- your default settings embedded
- maybe one optional field (like domain)

#### Examples:

`standard-topology`

A trait that patches the workload pod template with your org’s topology spread constraints. No user properties at all.

`standard-expose`

A trait that generates the HTTPRoute with your default:

- gatewayName
- listenerName
- path /
- service port

    Only property: domain (optional; could default to <component>.<namespace>.yourdns if you want).

`standard-hpa`

A trait that creates HPA with your default:

- min/max
- cpu threshold

    Optionally allow overrides.

`Why wrapper traits matter`: you don’t want app teams copying the same 25 lines of `topologySpreadConstraints` everywhere.

### 3. Auto-attach traits so teams don’t list them

If you want *really* minimal app YAML, make “standardization” happen automatically. Two common ways:

#### Option A: A “standardize” workflow step

Create a workflow step that:

- takes a component
- attaches your standard traits (standard-topology, standard-expose, standard-hpa)
- applies everything

Then apps can be as short as:

```YAML
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: demo
spec:
  components:
    - name: frontend
      type: golden-webservice
      properties:
        image: ghcr.io/acme/frontend:2026.02.14-1
        domain: frontend.example.com
  workflow:
    steps:
      - name: deploy
        type: standardize-and-apply
```

### Option B: Enforce defaults with policies

Policies are good for deployment behavior (apply-once, placement, override), but for “attach these traits,” workflow steps are usually more flexible.

### 4. Concrete “minimal lines” blueprint

**What app teams write**

Target: 10–15 lines total for a typical web app.

They provide:

- image
- domain (optional)
- maybe one scaling “profile” (small/medium/large)

**What the platform provides**

- `golden-webservice` component (includes probes/resources/service/envFrom)
- `standardize-and-apply` workflow step (attaches traits automatically)
- traits:
  - `standard-topology`
  - `standard-expose` (HTTPRoute defaults)
  - `standard-hpa`
- a default `apply-once` policy when autoscaling is enabled

### 5 Practical advice on what to bake where

**Bake into component defaults (almost always):**

- probes
- ports
- Service creation + naming
- resources baseline
- labels/selectors
- envFrom configMap naming convention (if you have one)

Keep as traits (because they’re environment-dependent or optional):

- topology spread constraints (sometimes varies by cluster)
- HTTPRoute/gateway attachment (varies by gateway controller and gateway n-]ame)
- HPA (not every workload wants it, and metrics differ)

Auto-attach via workflow step:

- topology + expose + HPA for “standard web apps”
skip expose if domain is empty (internal-only apps)

### 6. A very opinionated “profiles” trick (huge YAML reduction)

Instead of exposing every knob, expose a single parameter like:

```YAML
properties:
  image: ...
  profile: small
```

And in the component/traits you map profile → resources/probes/hpa defaults.
This is how you get maximum defaults without forcing app teams into giant YAMLs.
