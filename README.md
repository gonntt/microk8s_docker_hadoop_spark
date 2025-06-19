# microk8s_docker_hadoop_spark
ETAPA 1: Preparação das VMs OBS: ALGUNS NOMES ESTÃO COM O K3S, MAS DURANTE A REALIZAÇÃO DO LAB O SERVIÇO UTILIZADO FOI O MICROK8S PARA O KUBERNETS
1.1 Configuração no VirtualBox

Clone sua VM atual 2 vezes:

VM1: k3s-master (sua VM atual)
VM2: k3s-worker1 (clone 1)
VM3: k3s-worker2 (clone 2)


Configurar Rede em TODAS as 3 VMs:

VirtualBox → Configurações da VM → Rede
Adaptador 1: Placa em modo Bridge
Nome: Selecionar sua placa de rede física
✅ Avançado → Modo Promíscuo: Permitir Tudo

1.2 Configurar Hostnames e IPs
Em cada VM, execute:
bash# VM1 (Master)
sudo hostnamectl set-hostname k3s-master

# VM2 (Worker1) 
sudo hostnamectl set-hostname k3s-worker1

# VM3 (Worker2)
sudo hostnamectl set-hostname k3s-worker2

# Reiniciar para aplicar hostname
sudo reboot
Após reiniciar, em cada VM descubra o IP:
baship addr show | grep inet
# Anote os IPs, exemplo:
# Master: 192.168.1.100
# Worker1: 192.168.1.101  
# Worker2: 192.168.1.102
Configure o arquivo hosts em TODAS as VMs:
bashsudo nano /etc/hosts

# Adicione (substitua pelos IPs reais):
192.168.1.100 k3s-master
192.168.1.101 k3s-worker1
192.168.1.102 k3s-worker2
Teste a conectividade:
bash# Em cada VM, teste:
ping k3s-master
ping k3s-worker1
ping k3s-worker2

ETAPA 2: Instalação MicroK8s
2.1 Em TODAS as 3 VMs
bash# 1. Instalar MicroK8s
sudo snap install microk8s --classic

# 2. Adicionar usuário ao grupo
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube

# 3. Reiniciar sessão (importante!)
newgrp microk8s

# 4. Verificar instalação
microk8s status --wait-ready

# 5. Criar alias
echo 'alias kubectl=microk8s.kubectl' >> ~/.bashrc
source ~/.bashrc
2.2 No Master (VM1) - Configurar Cluster
bash# 1. Habilitar addons necessários
microk8s enable dns storage

# 2. Obter comando para adicionar nodes
microk8s add-node

# Vai aparecer algo como:
# microk8s join 192.168.1.100:25000/TOKEN
2.3 Nos Workers (VM2 e VM3)
bash# Execute o comando obtido no passo anterior
# Exemplo:
microk8s join 192.168.1.100:25000/SEU_TOKEN_AQUI
2.4 Verificar Cluster (no Master)
bashkubectl get nodes
# Deve mostrar os 3 nodes
ETAPA 3: Preparar Imagens (Igual ao K3s)
3.1 No Master - Construir e Exportar
bashcd ~/cluster-hadoop
sudo make build

# Exportar imagens
sudo docker save raulcsouza/spark-base-hadoop > /tmp/spark-base.tar
sudo docker save raulcsouza/spark-master-hadoop > /tmp/spark-master.tar
sudo docker save raulcsouza/spark-worker-hadoop > /tmp/spark-worker.tar
3.2 Copiar para Workers
bashscp /tmp/*.tar usuario@k3s-worker1:/tmp/
scp /tmp/*.tar usuario@k3s-worker2:/tmp/
3.3 Nos Workers - Importar
bashsudo docker load < /tmp/spark-base.tar
sudo docker load < /tmp/spark-master.tar
sudo docker load < /tmp/spark-worker.tar
ETAPA 4: Deploy Aplicação
4.1 Criar Arquivos Kubernetes (no Master)
bashmkdir -p ~/k8s-hadoop
cd ~/k8s-hadoop
4.2 Namespace
bashcat > namespace.yaml << 'EOF'
apiVersion: v1
kind: Namespace
metadata:
  name: hadoop-spark
EOF

kubectl apply -f namespace.yaml
4.3 Storage
bashsudo mkdir -p /opt/hadoop-data
sudo chmod 777 /opt/hadoop-data

cat > storage.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hadoop-data-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: microk8s-hostpath
  hostPath:
    path: /opt/hadoop-data
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: hadoop-data-pvc
  namespace: hadoop-spark
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
  storageClassName: microk8s-hostpath
EOF

kubectl apply -f storage.yaml
4.4 Spark Master
bashcat > spark-master.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spark-master
  namespace: hadoop-spark
spec:
  replicas: 1
  selector:
    matchLabels:
      app: spark-master
  template:
    metadata:
      labels:
        app: spark-master
    spec:
      hostname: spark-master
      containers:
      - name: spark-master
        image: raulcsouza/spark-master-hadoop
        imagePullPolicy: Never
        tty: true
        ports:
        - containerPort: 8080
        - containerPort: 7077
        - containerPort: 9870
        - containerPort: 8088
        - containerPort: 8888
        - containerPort: 4040
        - containerPort: 18080
        - containerPort: 8042
        - containerPort: 8000
        env:
        - name: SPARK_LOCAL_IP
          value: "spark-master"
        volumeMounts:
        - name: hadoop-data
          mountPath: /user_data
      volumes:
      - name: hadoop-data
        persistentVolumeClaim:
          claimName: hadoop-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: spark-master
  namespace: hadoop-spark
spec:
  selector:
    app: spark-master
  ports:
  - name: spark-ui
    port: 8080
    targetPort: 8080
    nodePort: 30080
  - name: spark-master-port
    port: 7077
    targetPort: 7077
  - name: hdfs-namenode
    port: 9870
    targetPort: 9870
    nodePort: 30870
  - name: yarn-ui
    port: 8088
    targetPort: 8088
    nodePort: 30088
  - name: jupyter
    port: 8888
    targetPort: 8888
    nodePort: 30888
  - name: spark-history
    port: 18080
    targetPort: 18080
    nodePort: 30180
  - name: api
    port: 8000
    targetPort: 8000
    nodePort: 30800
  - name: hdfs-port
    port: 9000
    targetPort: 9000
  type: NodePort
EOF

kubectl apply -f spark-master.yaml
4.5 Spark Workers
bashcat > spark-workers.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spark-worker
  namespace: hadoop-spark
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spark-worker
  template:
    metadata:
      labels:
        app: spark-worker
    spec:
      containers:
      - name: spark-worker
        image: raulcsouza/spark-worker-hadoop
        imagePullPolicy: Never
        tty: true
        ports:
        - containerPort: 8081
        - containerPort: 8042
        env:
        - name: SPARK_MASTER
          value: "spark://spark-master:7077"
        - name: SPARK_LOCAL_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: hadoop-data
          mountPath: /user_data
      volumes:
      - name: hadoop-data
        persistentVolumeClaim:
          claimName: hadoop-data-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: spark-worker
  namespace: hadoop-spark
spec:
  selector:
    app: spark-worker
  ports:
  - name: spark-worker-ui
    port: 8081
    targetPort: 8081
    nodePort: 30081
  - name: nodemanager
    port: 8042
    targetPort: 8042
  type: NodePort
EOF

kubectl apply -f spark-workers.yaml
ETAPA 5: Verificar e Testar
5.1 Verificar Pods
bashkubectl get pods -n hadoop-spark -w
# Aguardar todos ficarem "Running"
