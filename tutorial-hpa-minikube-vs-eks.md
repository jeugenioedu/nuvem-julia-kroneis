# Tutorial: Avaliação Comparativa de HPA — Minikube vs. Amazon EKS

Este tutorial cobre a implantação completa do experimento nos dois ambientes
exigidos pelo enunciado (Minikube local e AWS EKS na nuvem), usando os
**mesmos manifestos** (`manifests-hpa-demo.yaml`) para garantir uma
comparação justa.

Decisões já definidas para este projeto:
- **Imagem**: `registry.k8s.io/hpa-example` (oficial, sem necessidade de build)
- **Geração de carga**: Pod `busybox` em loop, rodando dentro do cluster
- **Exposição**: apenas interna (ClusterIP) — sem LoadBalancer, para reduzir custo e tempo no EKS

---

## Parte 0 — Pré-requisitos gerais

| Item | Necessário para |
|---|---|
| `kubectl` instalado | Ambos os ambientes |
| `minikube` instalado | Implantação A (Local) |
| Conta AWS pessoal (já configurada na Aula 6) | Implantação B (Nuvem) |
| `git` instalado | Versionamento e repositório GitHub |
| Repositório GitHub criado (público ou com acesso liberado aos docentes) | Entrega final |

Estrutura sugerida do repositório desde já:

```
hpa-project/
├── manifests/
│   └── manifests-hpa-demo.yaml
├── roteiros/
│   ├── roteiro-minikube.md
│   └── roteiro-eks.md
├── evidencias/
│   ├── minikube/
│   └── eks/
├── dados/
│   └── resultados-comparativos.csv
├── artigo/
│   └── artigo-hpa.pdf
└── README.md
```

Crie o repositório e faça o primeiro commit apenas com `manifests/manifests-hpa-demo.yaml` antes de prosseguir — assim cada etapa do tutorial pode virar um commit incremental, o que também é uma boa evidência de processo.

---

## Parte A — Implantação Local (Minikube)

### A.1 Iniciar o cluster Minikube

Um único node local, mas com CPU suficiente para comportar até 8 réplicas de 200m de limite (8 × 0.2 = 1.6 vCPU) mais overhead do sistema:

```bash
minikube start --cpus=4 --memory=4096
```

**Por que**: sem CPU suficiente alocada ao Minikube, os pods extras criados pelo HPA ficarão `Pending` por falta de capacidade — um problema de dimensionamento, não do HPA em si.

Confirme o node:

```bash
kubectl get nodes
kubectl describe node minikube | grep -A5 "Allocatable"
```

### A.2 Habilitar o Metrics Server

O HPA depende do Metrics Server para saber o uso real de CPU dos pods. No Minikube, isso é um addon:

```bash
minikube addons enable metrics-server
```

Aguarde e confirme que o Metrics Server está rodando:

```bash
kubectl get pods -n kube-system | grep metrics-server
```

Teste se as métricas já estão disponíveis (pode levar ~1 minuto para os primeiros dados aparecerem):

```bash
kubectl top nodes
```

Se retornar `error: metrics not available yet`, aguarde mais um pouco e tente novamente.

### A.3 Aplicar os manifestos

```bash
kubectl apply -f manifests/manifests-hpa-demo.yaml
```

Saída esperada:

```
namespace/hpa-demo created
deployment.apps/php-apache created
service/php-apache created
horizontalpodautoscaler.autoscaling/php-apache-hpa created
```

### A.4 Validar o estado inicial (linha de base)

Antes de gerar carga, capture o estado de repouso — **esta é a primeira evidência fotográfica**:

```bash
kubectl get deployment,pod,svc,hpa -n hpa-demo
kubectl top pods -n hpa-demo
```

Resultado esperado: 1 réplica, uso de CPU próximo de 0%, coluna `TARGETS` do HPA mostrando algo como `0%/50%` (não `<unknown>` — se aparecer `<unknown>`, o Metrics Server ainda não está fornecendo dados).

### A.5 Gerar carga (teste de estresse)

Em um terminal, inicie o `watch` do HPA para acompanhar em tempo real (deixe rodando em uma janela separada):

```bash
kubectl get hpa php-apache-hpa -n hpa-demo --watch
```

Em outro terminal, dispare o gerador de carga — um Pod `busybox` fazendo requisições HTTP em loop contra o Service:

```bash
kubectl run -i --tty load-generator \
  --rm --image=busybox:1.36 \
  --restart=Never \
  -n hpa-demo \
  -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

**Anote o horário exato (`date`) em que este comando foi iniciado** — será usado para calcular o tempo de reação do HPA no artigo.

### A.6 Observar o escalamento

Em uma terceira janela, monitore os pods sendo criados:

```bash
kubectl get pods -n hpa-demo --watch
```

E periodicamente:

```bash
kubectl top pods -n hpa-demo
```

**Capture evidências em pelo menos 3 momentos**: (1) logo após o CPU ultrapassar 50%, (2) durante o escalamento (réplicas subindo), (3) quando estabilizar no número máximo ou próximo dele.

Anote também, a partir da saída do `--watch` do HPA, o **horário em que `REPLICAS` mudou pela primeira vez** — a diferença para o horário anotado no passo A.5 é o "tempo de reação do HPA" que o artigo pede.

### A.7 Parar a carga e observar o scale-down

Interrompa o gerador de carga com `Ctrl+C` (o `--rm` remove o Pod automaticamente). Continue observando o HPA:

```bash
kubectl get hpa php-apache-hpa -n hpa-demo --watch
```

Como o manifesto define `stabilizationWindowSeconds: 60`, o número de réplicas deve começar a cair ~1 minuto após o fim da carga. Capture esta evidência também.

### A.8 Registrar os dados coletados

Preencha uma linha na planilha/CSV de resultados (ver Parte C):

- Horário de início da carga
- Horário do primeiro scale-up
- Número máximo de réplicas atingido
- Uso de CPU por pod durante o pico
- Horário de início do scale-down
- Tempo total até retornar a 1 réplica

### A.9 Limpeza do ambiente local

```bash
kubectl delete namespace hpa-demo
minikube stop
```

(Não precisa excluir o Minikube inteiro — apenas parar, já que não gera custo.)

---

## Parte B — Implantação em Nuvem (AWS EKS)

> Pré-requisito: cluster EKS ativo com managed node group (reaproveite a criação já feita na Aula 6, ou repita o mesmo processo com um novo cluster dedicado a este projeto — recomendo reaproveitar para economizar tempo, desde que o node group tenha capacidade suficiente).

### B.1 Verificar capacidade do node group

Com 8 réplicas de `limits.cpu: 200m` (1.6 vCPU no pico) mais os pods de sistema (`kube-system`), um único `t3.medium` (2 vCPU) fica no limite. Recomenda-se, para este experimento:

```bash
# No Console EKS, ou via CLI:
aws eks update-nodegroup-config \
  --cluster-name <nome-do-cluster> \
  --nodegroup-name <nome-do-nodegroup> \
  --scaling-config minSize=1,maxSize=2,desiredSize=2
```

**Por que**: evita que o teste seja limitado pela capacidade do node (o que mudaria o experimento de "teste de HPA" para "teste de falta de capacidade") — mas documente esse ajuste, pois ele também é um dado relevante para a seção de "facilidade de gerenciamento" do artigo (no Minikube isso foi um simples `--cpus=4`; no EKS, redimensionar o node group é uma operação AWS separada).

Confirme os nodes disponíveis:

```bash
kubectl get nodes
kubectl describe nodes | grep -A5 "Allocatable"
```

### B.2 Instalar o Metrics Server (manual, diferente do Minikube)

Diferente do Minikube, o EKS **não** vem com Metrics Server pré-instalado nem como addon habilitável em um comando — é necessário aplicar o manifesto oficial:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

Aguarde e confirme:

```bash
kubectl get deployment metrics-server -n kube-system
kubectl get pods -n kube-system | grep metrics-server
```

Teste:

```bash
kubectl top nodes
```

> **Ponto de diagnóstico comum no EKS**: se `kubectl top nodes` retornar erro de TLS/certificado, o Metrics Server pode precisar da flag `--kubelet-insecure-tls` (editar o Deployment `metrics-server` e adicionar o argumento). Isso normalmente não é necessário em clusters padrão do EKS, mas documente se ocorrer — é outro dado de comparação de "facilidade de gerenciamento".

### B.3 Aplicar os mesmos manifestos

```bash
kubectl apply -f manifests/manifests-hpa-demo.yaml
```

Mesma saída esperada da Parte A.3.

### B.4 a B.8 — Repetir os mesmos passos da Parte A

Execute exatamente os passos **A.4 a A.8** (linha de base → gerar carga → observar escalamento → parar carga → observar scale-down → registrar dados), agora no cluster EKS. Usar o **mesmo procedimento exato** é o que garante que a comparação entre os dois ambientes seja metodologicamente válida.

Diferenças esperadas a observar e registrar:
- Tempo de reação do HPA (pode variar por latência de rede até o Metrics Server / API do EKS)
- Tempo para o node group liberar capacidade, caso algum pod fique `Pending` temporariamente
- Qualquer comportamento diferente do Minikube

### B.9 Estimar custo aproximado

Registre, para o período em que os recursos ficaram ativos:

- Control plane EKS: cobrança por hora (consulte o valor vigente na página de pricing do EKS)
- Instâncias EC2 do node group (2× t3.medium, se ajustado no passo B.1): cobrança por hora conforme tipo de instância e região
- Volumes EBS associados aos nodes
- Não há Load Balancer neste experimento (decisão tomada: apenas carga interna), então não há esse custo

Use a **AWS Pricing Calculator** ou o **Cost Explorer** (se já houver dados de billing) e registre a data da consulta — valores de pricing mudam com o tempo.

### B.10 Limpeza completa (ordem importa)

Siga a mesma ordem já validada na Aula 6, adaptada para este projeto:

```bash
kubectl delete namespace hpa-demo
```

Depois, se o cluster foi criado exclusivamente para este projeto (não reaproveitado da Aula 6):

1. Remover o managed node group (Console EKS → Compute → node group → Delete)
2. Remover o cluster EKS (Console EKS → Delete cluster)
3. Remover a stack CloudFormation da VPC (se aplicável)
4. Remover as IAM roles criadas exclusivamente para este projeto

Se o cluster está sendo **reaproveitado** para outras atividades da disciplina, apenas reverta o node group ao tamanho original (passo B.1) em vez de excluí-lo, para não gerar custo desnecessário parado.

**Verifique no Billing/Cost Explorer** que nenhum recurso residual ficou ativo.

---

## Parte C — Consolidação dos dados para o artigo

### C.1 Tabela de resultados comparativos (preencher com os dados coletados)

| Métrica | Minikube | AWS EKS |
|---|---|---|
| Tempo de reação do HPA (carga → 1º scale-up) | | |
| Número máximo de réplicas atingido | | |
| Tempo até estabilizar no máximo | | |
| Uso médio de CPU por pod no pico | | |
| Tempo de scale-down (fim da carga → 1 réplica) | | |
| Complexidade de instalação do Metrics Server | Addon (1 comando) | Manifesto manual |
| Ajuste de capacidade necessário | `minikube start --cpus` | Redimensionar node group |
| Custo aproximado do experimento | R$ 0 (local) | (preencher) |

### C.2 Estrutura sugerida do artigo (mapeando com o que foi coletado)

1. **Introdução**: contextualizar autoescalamento em nuvem, diferença entre escalamento vertical/horizontal, papel do HPA
2. **Arquitetura e topologia**: usar os diagramas de fluxo já documentados (ex.: os do tutorial da Aula 6, adaptados) para Minikube e EKS — deixar explícita a diferença de node único local vs. managed node group gerenciado pela AWS
3. **Metodologia**: descrever o manifesto único (`manifests-hpa-demo.yaml`), a estratégia de carga (busybox loop) e o motivo de manter o mesmo procedimento nos dois ambientes
4. **Resultados**: usar a tabela da seção C.1 e os gráficos de `kubectl top pods` ao longo do tempo (se quiserem, podem plotar os números anotados manualmente em um gráfico simples)
5. **Análise comparativa**: discutir os motivos das diferenças observadas (latência de rede, isolamento de recursos, gerenciamento manual vs. addon)
6. **Conclusão**: quando cada abordagem é preferível (desenvolvimento local vs. produção), limitações do experimento (single-node, apenas métrica de CPU, sem teste de HA entre zonas)

### C.3 Checklist de evidências fotográficas (mínimo 12, conforme sugerido)

- [ ] Estado inicial (1 réplica) — Minikube
- [ ] Estado inicial (1 réplica) — EKS
- [ ] HPA com métricas ativas (`kubectl get hpa`) antes da carga — ambos
- [ ] Pico de CPU durante a carga — ambos
- [ ] Escalamento em progresso (`kubectl get pods --watch`) — ambos
- [ ] Número máximo de réplicas atingido — ambos
- [ ] Scale-down em progresso — ambos
- [ ] Metrics Server rodando — ambos (destacar a diferença de instalação)
- [ ] Nodes e capacidade (`kubectl describe node`) — ambos
- [ ] Custo estimado (Console AWS / Cost Explorer) — apenas EKS
- [ ] Limpeza final confirmada — ambos

---

## Observações finais

- Cada aluno do grupo deve executar **sua própria** implantação individual em ambos os ambientes (exigência do enunciado), mesmo usando os manifestos compartilhados do repositório.
- Mantenha o `manifests-hpa-demo.yaml` idêntico entre os ambientes — qualquer diferença necessária (como o `stabilizationWindowSeconds`) deve ser documentada, não alterada silenciosamente, para não comprometer a comparação.
- Lembre-se do prazo de remoção dos recursos AWS no mesmo dia da execução, como já praticado na Aula 6.
