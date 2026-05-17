# Gerenciando Instâncias EC2 na AWS — Desafio DIO

> Repositório: [github.com/sir-devtech/Desafio-AWS-](https://github.com/sir-devtech/Desafio-AWS-)

## Sobre o Projeto

Este repositório documenta minha experiência prática com o Amazon EC2, aplicada em um cenário real: o deploy de um **sistema de GCM (Gerenciamento de Clientes)** na nuvem AWS, com aplicativo móvel integrado.

O projeto foi executado durante os **6 meses do nível gratuito da AWS (Free Tier)**, com o sistema em produção, pronto para uso e disponível para demonstração a potenciais clientes.

> Dois ambientes distintos foram mantidos: um para o município de **Ipero/SP** e um **ShowRoom** de demonstração.

---

## Contexto Real de Uso

| Item | Detalhe |
|------|---------|
| **Sistema** | GCM — Guarda Civil Municipal |
| **Funcionalidades** | Mural Eletrônico, Banco de Horas, Viaturas, BOGCM, Despachos, Fiscalização, Botão do Pânico, Rastreamento |
| **Ambiente** | Produção / Demonstração para prefeituras |
| **Período ativo** | 6 meses (AWS Free Tier) |
| **Região** | América do Sul — São Paulo (`sa-east-1`) |
| **Status atual** | Standby via AMI + Snapshot EBS |

O sistema foi mantido rodando na AWS com uma instância EC2, banco de dados e aplicativo funcional — pronto para demonstração a prefeituras. Como o fechamento com o cliente não ocorreu no período, optei por **encerrar a instância e criar AMIs/Snapshots EBS** para preservar todo o ambiente, possibilitando restauração futura sem perda de dados.

### Sistema em Funcionamento

![Dashboard do Sistema GCM](Captura%20de%20tela%202026-05-17%20195440.png)

> Dashboard principal com Mural Eletrônico, controle de Viaturas Ativas, Banco de Horas dos agentes, Ordens de Serviço e rastreamento em tempo real.

---

## Arquitetura Utilizada

```
┌─────────────────────────────────────────────────────┐
│                    AWS Cloud                        │
│                                                     │
│   [Cliente / S3]  ──►  [EC2 Instance]  ──►  [EBS]  │
│       (Upload)         (Aplicação GCM)   (Volume)   │
│                              │                      │
│                         [RDS / DB]                  │
│                                                     │
└─────────────────────────────────────────────────────┘
         ▲
    [App Mobile]
    (Usuário final)
```

**Componentes principais:**
- **EC2** — Servidor da aplicação GCM (backend + API)
- **EBS (Elastic Block Store)** — Volume de armazenamento da instância
- **S3** — Armazenamento de arquivos e assets do sistema
- **AMI** — Imagem gerada para facilitar reinstanciação futura
- **Snapshots EBS** — Backup do estado do sistema para standby

---

## Estratégia de Snapshots — Standby por Cliente

Quando um projeto entra em fase de negociação sem fechamento imediato, a estratégia utilizada foi:

### 1. Criar Snapshot do Volume EBS
```
EC2 Console → Elastic Block Store → Volumes
→ Selecionar volume → Actions → Create Snapshot
→ Descrever: "GCM-sistema-standby-[data]"
```

### 2. Criar AMI a partir da instância
```
EC2 Console → Instances → Selecionar instância
→ Actions → Image and templates → Create image
→ Nome: "GCM-AMI-v1"
```

### 3. Parar/Terminar a instância
```
→ Actions → Instance State → Stop (para pausar cobrança de computação)
```

> **Benefício:** O Snapshot EBS mantém o estado completo do sistema. Quando o cliente fechar, basta restaurar o volume ou lançar uma nova instância a partir da AMI — sem reconfiguração do zero.

---

## Painel EC2 — Estado Atual

No console AWS (`sa-east-1`):

- **Instâncias em execução:** 0 (em standby)
- **AMIs criadas:** 2 (ambientes completos preservados)
- **Snapshots EBS:** 2 (volumes preservados — ~7.6 GiB cada)
- **Par de chaves:** 1 (acesso SSH mantido)
- **Grupos de segurança:** 2 (configurações preservadas)
- **Custo acumulado:** $0.62 (apenas storage das AMIs/snapshots)

### AMIs Registradas

| Nome | AMI ID | Criação | Plataforma | Status |
|------|--------|---------|------------|--------|
| `Ipero_Backup` | ami-05662ab021ebea673 | 02/03/2026 09:09 GMT-3 | Linux/UNIX (EBS) | ✅ Disponível |
| `ShowRoom_Backup` | ami-0d0067dbb25e480a7 | 02/03/2026 09:10 GMT-3 | Linux/UNIX (EBS) | ✅ Disponível |

- **`Ipero_Backup`** — ambiente completo do sistema GCM configurado para o município de Ipero/SP
- **`ShowRoom_Backup`** — ambiente de demonstração para apresentação a novos clientes

![AMIs EC2 — ambientes em standby](Captura%20de%20tela%202026-05-17%20195239.png)

### Snapshots EBS

| Snapshot ID | Tamanho usado | Volume total | Origem | Status |
|-------------|--------------|--------------|--------|--------|
| snap-0dc442ca0cabeb824 | 7.6 GiB | 30 GiB | CreateImage (Ipero_Backup) | ✅ Concluído |
| snap-0fc1a92b8b016932f | 7.57 GiB | 30 GiB | CreateImage (ShowRoom_Backup) | ✅ Concluído |

> Os snapshots foram gerados automaticamente durante a criação das AMIs, garantindo cópia fiel do estado dos volumes EBS.

![Snapshots EBS — backups dos ambientes](Captura%20de%20tela%202026-05-17%20195849.png)

---

## Diagrama de Arquitetura — Fluxo Completo

### Arquitetura do Sistema GCM

```
[Usuário / App]
      │
      ▼
[S3 — Upload de Arquivo]
      │
      ▼
[EC2 — Aplicação GCM]  ◄──► [EBS — Armazenamento]
      │
      ▼
[RDS — Banco de Dados]
```

### Arquitetura de Processamento (Lambda)

```
[Sistema de Arquivos]
      │
      ▼
[Arquivo] ──► [S3] ──Trigger──► [Lambda Function] ──► [DynamoDB]
                                (Node.js / Python)
```

---

## Lições Aprendidas

### Gestão de Custos
- O Free Tier cobre 750h/mês de instância `t2.micro` durante 12 meses
- Snapshots EBS têm custo por GB/mês — ideal para standby de longo prazo
- Parar a instância elimina custo de computação, mas mantém cobrança do EBS

### Snapshots vs AMI
| | **Snapshot EBS** | **AMI** |
|---|---|---|
| O que salva | Volume de dados | Volume + configuração da instância |
| Restauração | Attach a nova instância | Lançar nova instância direto |
| Custo | Por GB armazenado | Por GB dos snapshots subjacentes |
| Ideal para | Backup de dados | Replicar ambiente completo |

### Boas Práticas Aplicadas
- Snapshot antes de qualquer alteração crítica
- AMI gerada com o sistema configurado e testado
- Par de chaves SSH preservado com segurança
- Security Groups documentados para restauração rápida

---

## Como Restaurar o Ambiente

Se um cliente fechar contrato:

```bash
# 1. EC2 Console → AMIs → Selecionar "Ipero_Backup" ou "ShowRoom_Backup"
# 2. Actions → Launch instance from AMI
# 3. Selecionar tipo de instância (t2.micro para Free Tier)
# 4. Associar par de chaves existente
# 5. Configurar Security Group existente
# 6. Launch → sistema restaurado em minutos, exatamente como estava
```

| Ambiente | AMI | Uso |
|----------|-----|-----|
| Ipero/SP | `Ipero_Backup` | Sistema configurado para o cliente municipal |
| Demonstração | `ShowRoom_Backup` | Apresentação a novos prospects |

---

## Recursos Utilizados

- [Amazon EC2 — Documentação](https://docs.aws.amazon.com/ec2/)
- [Amazon EBS Snapshots](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EBSSnapshots.html)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [Criação de AMIs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html)
- [draw.io — Diagramas de Arquitetura](https://www.drawio.com)

---

## Conclusão

Este projeto demonstra um caso de uso real e prático do EC2: não apenas como laboratório, mas como infraestrutura produtiva para um sistema comercial. A estratégia de **Snapshots + AMI para standby** é uma abordagem profissional e econômica para projetos em fase de negociação, eliminando custos operacionais enquanto preserva 100% do ambiente para retomada imediata.
=======
# Desafio-AWS-
