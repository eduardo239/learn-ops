# 🚀 Learn Ops — Do Básico à Prática em DevOps & Cloud

<div align="center">

[![Blog](https://img.shields.io/badge/Acesse_o_Blog-Learn--Ops-FF5722?style=for-the-badge&logo=blogger&logoColor=white)](https://learn-ops.blogspot.com/)
[![GitHub repo](https://img.shields.io/badge/GitHub-Repo-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/eduardo239/learn-ops)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Minikube](https://img.shields.io/badge/Minikube-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://minikube.sigs.k8s.io/)

<p align="center">
  <b>Repositório de estudos, guias práticos, laboratórios e referências em DevOps, Cloud Native, Containers e Infraestrutura.</b>
</p>

🔗 **Acesse todos os artigos, publicações e conteúdos aprofundados no blog oficial:**  
👉 **[https://learn-ops.blogspot.com/](https://learn-ops.blogspot.com/)** 👈

---

</div>

## 📌 Sobre o Projeto

O **Learn Ops** é uma iniciativa voltada ao aprendizado contínuo, compartilhamento de conhecimento e documentação prática no ecossistema **DevOps, SRE e Cloud Native**.

Este repositório funciona como o braço técnico e prático do blog [Learn Ops](https://learn-ops.blogspot.com/), reunindo:
- 📖 **Guias passo a passo e apostilas completas** (PDF e HTML interativo);
- 🛠️ **Manifestos, scripts e arquivos de configuração** prontos para laboratórios;
- 🧪 **Laboratórios práticos (Hands-on)** para simulação de cenários reais;
- 📐 **Boas práticas de arquitetura, segurança, operação e automação**.

---

## 🌐 Blog Oficial — Learn Ops

Para explicações conceituais, análises de mercado, novos tutoriais e discussões detalhadas, visite o nosso blog:

<div align="center">

### 🌐 [learn-ops.blogspot.com](https://learn-ops.blogspot.com/)

*Artigos sobre Cultura DevOps, Ferramentas, Automação, Containers, CI/CD e Cloud.*

</div>

---

## 📂 Conteúdo Disponível no Repositório

### ☸️ Trilha Kubernetes: Guia Completo com Minikube (`/minikube`)

Material de estudo para quem deseja dominar o Kubernetes na prática a partir de um ambiente local com **Minikube**. Disponível tanto em formato para leitura offline (**PDF**) quanto em visualizador web estruturado (**HTML**).

📁 **Caminho:** [`/minikube`](./minikube)

#### 📚 Estrutura dos Módulos do Guia:

| Módulo | Tema Principal | Descrição dos Conceitos Abordados |
| :--- | :--- | :--- |
| **Parte 1** | **Fundamentos** | Conceitos essenciais de containers, arquitetura do Kubernetes (Control Plane, Worker Nodes, etcd, kube-apiserver). |
| **Parte 2** | **Ambiente com Minikube** | Instalação, configuração de drivers (Docker/KVM), cluster inicial, addons e métricas locais. |
| **Parte 3** | **kubectl, Pods e Organização** | Comandos essenciais do `kubectl`, ciclo de vida de Pods, Namespaces, Labels e Selectors. |
| **Parte 4** | **Deployments, Services e Ingress** | Estratégias de Deploy (Rolling Update), Services (`ClusterIP`, `NodePort`, `LoadBalancer`) e roteamento HTTP com Ingress. |
| **Parte 5** | **Configuração e Armazenamento** | Desacoplamento com `ConfigMaps` e `Secrets`, persistência de dados com `PV` (Persistent Volumes) e `PVC`. |
| **Parte 6** | **Operação, Escala e Segurança** | Auto-escalabilidade horizontal (`HPA`), recursos (`Requests` & `Limits`), probes (`Liveness`/`Readiness`) e RBAC básico. |
| **Parte 7** | **Troubleshooting e Projeto Final** | Diagnóstico e debug (`logs`, `describe`, `exec`), cenário hands-on completo integrando todos os componentes e cheatsheet de referência. |

#### 📥 Arquivos para Download e Leitura:
- 📄 **PDF:** [`minikube/Kubernetes com Minikube — Guia completo.pdf`](./minikube/Kubernetes%20com%20Minikube%20%E2%80%94%20Guia%20completo.pdf)
- 🌐 **HTML:** [`minikube/Kubernetes com Minikube — Guia completo.html`](./minikube/Kubernetes%20com%20Minikube%20%E2%80%94%20Guia%20completo.html)

---

## 🗺️ Roadmap de Conteúdo

O repositório e o blog continuam em evolução. Os próximos tópicos em desenvolvimento incluem:

- [x] **Kubernetes com Minikube** (Guia definitivo e labs)
- [ ] **Docker & Containerização Avançada** (Multi-stage builds, rootless containers, otimização de imagens)
- [ ] **Infraestrutura como Código (IaC)** (Terraform, OpenTofu, Ansible)
- [ ] **Pipelines de CI/CD** (GitHub Actions, GitLab CI, ArgoCD para GitOps)
- [ ] **Observabilidade & Monitoramento** (Prometheus, Grafana, OpenTelemetry)
- [ ] **Cloud Providers** (GCP, AWS e Azure labs práticos)

---

## 🛠️ Como Utilizar este Repositório

1. **Clone o repositório:**
   ```bash
   git clone https://github.com/eduardo239/learn-ops.git
   cd learn-ops
   ```

2. **Explore os materiais:**
   - Para abrir o guia HTML localmente no seu navegador:
     ```bash
     xdg-open "minikube/Kubernetes com Minikube — Guia completo.html" # No Linux
     open "minikube/Kubernetes com Minikube — Guia completo.html"     # No macOS
     ```
   - Ou abra o arquivo PDF em seu visualizador de documentos preferido.

3. **Acompanhe os artigos complementares:**
   - Acesse o blog [Learn Ops](https://learn-ops.blogspot.com/) para acompanhar discussões e novos lançamentos.

---

## 🤝 Contribuições

Contribuições, correções, sugestões de novos temas e melhorias são sempre bem-vindas!

1. Faça um **Fork** do projeto
2. Crie uma branch para sua feature/correção (`git checkout -b feature/novo-topico`)
3. Commit suas alterações (`git commit -m 'feat: Adiciona lab sobre ingress controllers'`)
4. Faça o push para a branch (`git push origin feature/novo-topico`)
5. Abra um **Pull Request**

---

## 📬 Contato e Redes

- 🌐 **Blog:** [https://learn-ops.blogspot.com/](https://learn-ops.blogspot.com/)
- 💻 **GitHub:** [@eduardo239](https://github.com/eduardo239)

---

<div align="center">

Feito com dedicação para a comunidade **DevOps & Cloud** ☕  
*Aprenda, pratique e compartilhe conhecimento no [Learn Ops](https://learn-ops.blogspot.com/)!*

</div>
