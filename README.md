# TP-Orchestration-de-Services-R-seau



## 4. Étapes de réalisation

### 4.1 Déploiement conteneurs Docker
**Fichier `docker-compose.yml` :**

```yaml
services:
  firewall:
    image: wallarm/api-firewall
    container_name: vnf-firewall
    environment:
      - AFW_SERVER_LISTEN=8080
      - AFW_SERVER_ENDPOINT=http://vnf-lb:80
      - AFW_REQUEST_VALIDATION=false
      - AFW_SERVER_REPORT_ONLY=true
    ports:
      - "8080:8080"
    networks:
      - net-chain

  loadbalancer:
    image: haproxytech/haproxy-alpine
    container_name: vnf-lb
    ports:
      - "9090:80"
    networks:
      - net-chain

  webserver:
    image: nginx:alpine
    container_name: vnf-web
    ports:
      - "8081:80"
    networks:
      - net-chain

networks:
  net-chain:
    driver: bridge

```



* **Firewall** : point d'entrée unique de la chaîne de services. Il analyse le trafic entrant pour bloquer les requêtes malveillantes avant qu'elles n'atteignent l'infrastructure interne.
* **Load Balancer** : agit comme un orchestrateur. Distribue le trafic réseau entre plusieurs serveurs finaux pour éviter la saturation d'un seul point.
* **Web Server** :  Il héberge l'application.

### 4.1 Modélisation TOSCA

**Fichier `service-chain.yaml` :**

```yaml
tosca_definitions_version: tosca_simple_yaml_1_3

node_types:
  vnf_node:
    derived_from: tosca.nodes.Root

topology_template:
  node_templates:
    firewall:
      type: vnf_node
      interfaces:
        Standard:
          operations:
            create: playbooks/deploy_firewall.yaml

    loadbalancer:
      type: vnf_node
      requirements:
        - dependency: firewall
      interfaces:
        Standard:
          operations:
            create: playbooks/deploy_loadbalancer.yaml

    webserver:
      type: vnf_node
      requirements:
        - dependency: loadbalancer
      interfaces:
        Standard:
          operations:
            create: playbooks/deploy_webserver.yaml
```

Le modèle TOSCA structure notre chaîne de services. L’orchestrateur xOpera construit un Graphe Dirigé Acyclique pour ordonnancer le déploiement. Il interprète ces liens comme des contraintes de séquençage strictes, interdisant le lancement simultané des VNFs.
