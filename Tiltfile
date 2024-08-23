# -*- mode: Python -*-

tilt_settings_file = "./tilt-settings.yaml"
settings = read_yaml(tilt_settings_file)

kubectl_cmd = "kubectl"

# verify kubectl command exists
if str(local("command -v " + kubectl_cmd + " || true", quiet = True)) == "":
    fail("Required command '" + kubectl_cmd + "' not found in PATH")

# Install cert manager
load('ext://cert_manager', 'deploy_cert_manager')
deploy_cert_manager()

# Create the kubewarden namespace
# This is required since the helm() function doesn't support the create_namespace flag
load('ext://namespace', 'namespace_create')
namespace_create('kubewarden')

# Install CRDs

# Install the policy report CRDs
k8s_yaml(settings.get('audit_scanner_path') + '/config/crd/wgpolicyk8s.io_clusterpolicyreports.yaml')
k8s_yaml(settings.get('audit_scanner_path') + '/config/crd/wgpolicyk8s.io_policyreports.yaml')

crd = kustomize('config/crd')
k8s_yaml(crd)
roles = decode_yaml_stream(kustomize('config/rbac'))
cluster_rules = []
namespace_rules = []
roles_rules_mapping = {
	"ClusterRole": {},
	"Role": {},
}

for role in roles:
    if role.get('kind') == 'ClusterRole':
	roles_rules_mapping["ClusterRole"][role.get('metadata').get('name')] = role.get('rules')
    elif role.get('kind') == 'Role':
        roles_rules_mapping["Role"][role.get('metadata').get('name')] = role.get('rules')

if len(roles_rules_mapping["ClusterRole"]) == 0 or len(roles_rules_mapping["Role"]) == 0:
    fail("Failed to load cluster and namespace roles")

# Get the webhook configuration with the latest values generated in the 
# controller repository. 
mutating_webhooks = []
validating_webhooks = []
webhooks_config = decode_yaml_stream(kustomize('config/webhook'))
for webhook_config in webhooks_config:
 if webhook_config.get('kind') == 'MutatingWebhookConfiguration':
	 mutating_webhooks = webhook_config.get('webhooks')
	 for i in range(len(mutating_webhooks)):
	     mutating_webhooks[i]["clientConfig"]["service"]["name"] = "kubewarden-controller-webhook-service"
	     mutating_webhooks[i]["clientConfig"]["service"]["namespace"] = "kubewarden"
 if webhook_config.get('kind') == 'ValidatingWebhookConfiguration':
	 validating_webhooks = webhook_config.get('webhooks')
	 for i in range(len(validating_webhooks)):
	     validating_webhooks[i]["clientConfig"]["service"]["name"] = "kubewarden-controller-webhook-service"
	     validating_webhooks[i]["clientConfig"]["service"]["namespace"] = "kubewarden"


# Install kubewarden-controller helm chart
install = helm(
    settings.get('helm_charts_path') + '/charts/kubewarden-controller/', 
    name='kubewarden-controller', 
    namespace='kubewarden', 
    set=['image.repository=' + settings.get('image'), 'global.cattle.systemDefaultRegistry=' + settings.get('registry')]
)

objects = decode_yaml_stream(install)
for o in objects:
    # Update the root security group. Tilt requires root access to update the
    # running process.
    if o.get('kind') == 'Deployment' and o.get('metadata').get('name') == 'kubewarden-controller':
        o['spec']['template']['spec']['securityContext']['runAsNonRoot'] = False
        # Disable the leader election to speed up the startup time.
        o['spec']['template']['spec']['containers'][0]['args'].remove('--leader-elect')
	# Enable policy groups feature
	envvars = o['spec']['template']['spec']['containers'][0].get('env', [])
	envvars.append({'name': 'KUBEWARDEN_ENABLE_POLICY_GROUPS', 'value': 'true'})
	o['spec']['template']['spec']['containers'][0]['env']  = envvars

    # Update the cluster and namespace roles used by the controller. This ensures
    # that always we have the latest roles applied to the cluster.
    if o.get('kind') == 'ClusterRole' and o.get('metadata').get('name') == 'kubewarden-controller-manager-cluster-role':
	o['rules'] = roles_rules_mapping["ClusterRole"]["manager-role"]
    if o.get('kind') == 'Role' and o.get('metadata').get('name') == 'kubewarden-controller-manager-namespaced-role':
	o['rules'] = roles_rules_mapping["Role"]["manager-role"]
    if o.get('kind') == 'Role' and o.get('metadata').get('name') == 'kubewarden-controller-leader-election-role':
	o['rules'] = roles_rules_mapping["Role"]["leader-election-role"]

    # Update the webhook configuration with the latest values generated in the 
    # controller repository. This useful when adding/updating webhooks.
    if o.get('kind') == 'MutatingWebhookConfiguration':
	o['webhooks'] = mutating_webhooks
    if o.get('kind') == 'ValidatingWebhookConfiguration':
	o['webhooks'] = validating_webhooks

updated_install = encode_yaml_stream(objects)
k8s_yaml(updated_install)

# enable hot reloading by doing the following:
# - locally build the whole project
# - create a docker imagine using tilt's hot-swap wrapper
# - push that container to the local tilt registry
# Once done, rebuilding now should be a lot faster since only the relevant
# binary is rebuilt and the hot swat wrapper takes care of the rest.
local_resource(
    'manager',
    "CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o bin/manager cmd/main.go",
    deps = [
        "go.mod",
        "go.sum",
        "cmd",
        "internal",
        "api",
    ],
)

# Build the docker image for our controller. We use a specific Dockerfile
# since tilt can't run on a scratch container.
entrypoint = ['/manager', '-zap-devel']
dockerfile = 'tilt.dockerfile'

load('ext://restart_process', 'docker_build_with_restart')
docker_build_with_restart(
    settings.get('registry') + '/' + settings.get('image'),
    '.',
    dockerfile = dockerfile,
    entrypoint = entrypoint,
    # `only` here is important, otherwise, the container will get updated
    # on _any_ file change.
    only=[
      './bin',
    ],
    live_update = [
        sync('./bin/manager', '/manager'),
    ],
)

