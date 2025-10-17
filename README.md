# from your project folder (with the Vagrantfile):
vagrant up --provider=virtualbox

# log into the control-plane:
vagrant ssh cp1
kubectl get nodes -o wide
kubectl get pods -A

# on your Mac in the same folder:
vagrant ssh cp1 -c "sudo cat /etc/kubernetes/admin.conf" > admin.conf
# ensure it points at 192.168.56.10 (it should already)
sed -n 's/server: /server: /p' admin.conf
export KUBECONFIG="$(pwd)/admin.conf"
kubectl get nodes
