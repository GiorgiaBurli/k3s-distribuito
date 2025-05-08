# Relazione Tecnica: Calcolo Distribuito con K3s su Azure

## Obiettivo dell'esercizio

Creare un'infrastruttura composta da almeno due macchine virtuali (una master, una agent), installare e configurare K3s e verificare che il calcolo venga distribuito su entrambi i nodi (pod schedulati sia su master che agent).

---

## Preparazione infrastruttura con Terraform (da PC locale - PowerShell)

### 1. Creazione dei file:
- `main.tf`: definisce l'infrastruttura Azure (VM, subnet, NSG, IP, NIC ecc.)
- `variables.tf`: contiene le variabili (nome risorse, location, user ecc.)
- `outputs.tf`: stampa gli IP pubblici

### 2. Comandi eseguiti (da PowerShell nella cartella Terraform):
```powershell
terraform init
terraform plan
terraform apply
```

> Risultato: vengono create due VM (master + agent), ognuna con IP pubblico e SSH abilitato.

---

## Accesso via SSH alle VM

### Comandi da PowerShell:
```powershell
ssh -i C:\Users\user\.ssh\id_ed25519 azureuser@<IP_MASTER>
ssh -i C:\Users\user\.ssh\id_ed25519 azureuser@<IP_AGENT>
```

---

## Installazione K3s

### Sulla VM master:
```bash
curl -sfL https://get.k3s.io | sh -
sudo k3s kubectl get node
```

### Recupero token e IP privato:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
ip a show eth0
```

### Sulla VM agent:
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<IP_PRIVATO_MASTER>:6443 K3S_TOKEN=<TOKEN> sh -
```

### Verifica da VM master:
```bash
sudo k3s kubectl get nodes
```
> Il risultato deve mostrare sia master che agent in stato `Ready`

---

## Test distribuzione: Parallel Job Kubernetes

### Sulla VM master:

#### Creazione file `job.yaml`:
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  parallelism: 3
  completions: 3
  template:
    spec:
      containers:
      - name: test
        image: busybox
        command: ["sh", "-c", "echo Hello from $(hostname); sleep 30"]
      restartPolicy: Never
```

```bash
nano job.yaml
sudo k3s kubectl apply -f job.yaml
sudo k3s kubectl get pods -o wide
```

> L'output mostrerà i pod in esecuzione sia su master che su agent, dimostrando la distribuzione del carico.

---

## Conclusione

- Infrastruttura su Azure creata correttamente con Terraform
- K3s installato con successo su due VM
- Agent registrato correttamente al master
- Verificato che i pod vengono distribuiti su più VM
- Eseguito test di carico distribuito con parallel job
