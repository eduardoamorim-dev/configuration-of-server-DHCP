# RELATÓRIO TÉCNICO: CONFIGURAÇÃO DE SERVIDOR DHCP

**Disciplina:** Laboratório de Redes  
**Professor:** Júnio Moreira  
---

## RESUMO EXECUTIVO

Este relatório documenta a implementação de um servidor DHCP em ambiente virtualizado, configurado para atender duas sub-redes distintas (administrativa e alunos) e fornecer endereçamento IP fixo para uma estação específica (CWIN).

---

## 1. OBJETIVOS

- Configurar servidor DHCP para distribuição automática de endereços IP em duas redes distintas
- Implementar reserva de IP fixo para estação de trabalho específica
- Validar o funcionamento através de testes práticos
- Documentar todo o processo de configuração

---

## 2. ARQUITETURA DA SOLUÇÃO

### 2.1 Topologia de Rede

**Servidor DHCP:**
- Sistema Operacional: Ubuntu Server 24.04 LTS
- Interfaces de rede: 3 adaptadores
  - enp0s3: NAT (acesso à internet) - 10.0.2.15/24
  - enp0s8: Rede Administrativa - 172.16.1.1/24
  - enp0s9: Rede de Alunos - 192.168.1.1/24
- RAM: 2048 MB
- Disco: 20 GB


**Cliente CWIN:**
- Sistema Operacional: Windows 10 (64-bit)
- Interface: Rede Administrativa
- IP Fixo Atribuído: 172.16.1.16
- MAC Address: 08:00:27:EF:FD:0F

### 2.2 Faixas de Endereçamento

| Rede | Endereço de Rede | Gateway | Faixa DHCP | Máscara |
|------|------------------|---------|------------|---------|
| Administrativa | 172.16.1.0 | 172.16.1.1 | 172.16.1.10 - 172.16.1.100 | 255.255.255.0 |
| Alunos | 192.168.1.0 | 192.168.1.1 | 192.168.1.10 - 192.168.1.200 | 255.255.255.0 |

**DNS Servers:** 8.8.8.8, 8.8.4.4 (Google Public DNS)

---

## 3. PROCEDIMENTOS DE CONFIGURAÇÃO

### 3.1 Preparação do Ambiente Virtual

**Criação da VM Servidor:**
1. VirtualBox → Novo
2. Nome: Servidor-DHCP
3. Tipo: Linux / Ubuntu (64-bit)
4. Memória: 2048 MB
5. Disco: 20 GB (dinamicamente alocado)

**Configuração de Rede:**
- Adaptador 1: NAT (internet)
- Adaptador 2: Rede interna "rede-administrativa"
- Adaptador 3: Rede interna "rede-alunos"

**Instalação do Sistema:**
- Ubuntu Server 24.04 LTS
- Instalação padrão
- OpenSSH Server habilitado para administração remota

### 3.2 Configuração das Interfaces de Rede

**Arquivo:** `/etc/netplan/50-cloud-init.yaml`

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      addresses:
        - 172.16.1.1/24
      dhcp4: false
    enp0s9:
      addresses:
        - 192.168.1.1/24
      dhcp4: false
```

**Aplicação da configuração:**
```bash
sudo netplan apply
```

**Verificação:**
```bash
ip addr show
```

### 3.3 Instalação do Servidor DHCP

```bash
sudo apt update
sudo apt install isc-dhcp-server -y
```

### 3.4 Configuração do Servidor DHCP

**Arquivo:** `/etc/default/isc-dhcp-server`

```bash
INTERFACESv4="enp0s8 enp0s9"
INTERFACESv6=""
```

**Arquivo:** `/etc/dhcp/dhcpd.conf`

```bash
# Configurações globais
default-lease-time 600;
max-lease-time 7200;
authoritative;

# Rede Administrativa (172.16.1.0/24)
subnet 172.16.1.0 netmask 255.255.255.0 {
    range 172.16.1.10 172.16.1.100;
    option routers 172.16.1.1;
    option subnet-mask 255.255.255.0;
    option domain-name-servers 8.8.8.8, 8.8.4.4;
    option domain-name "admin.local";
    default-lease-time 600;
    max-lease-time 7200;
}

# Rede de Alunos (192.168.1.0/24)
subnet 192.168.1.0 netmask 255.255.255.0 {
    range 192.168.1.10 192.168.1.200;
    option routers 192.168.1.1;
    option subnet-mask 255.255.255.0;
    option domain-name-servers 8.8.8.8, 8.8.4.4;
    option domain-name "students.local";
    default-lease-time 600;
    max-lease-time 7200;
}

# Reserva de IP Fixo para CWIN
host CWIN {
    hardware ethernet 08:00:27:EF:FD:0F;
    fixed-address 172.16.1.16;
}
```

### 3.5 Inicialização do Serviço

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl enable isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

### 3.6 Configuração de Acesso Remoto (SSH)

**Port Forwarding configurado no VirtualBox:**
- Host Port: 2222
- Guest Port: 22
- Protocolo: TCP

**Acesso remoto:**
```bash
ssh -p 2222 usuario@localhost
```

---

## 4. VALIDAÇÃO E TESTES

### 4.1 Verificação do Servidor

**Status do serviço DHCP:**
```bash
sudo systemctl status isc-dhcp-server
```
**Resultado esperado:** active (running)

**Interfaces configuradas:**
```bash
ip addr show
```
**Resultado esperado:**
- enp0s8: 172.16.1.1/24
- enp0s9: 192.168.1.1/24

### 4.2 Teste do IP Fixo - CWIN

**Configuração da VM Cliente:**
- Nome: CWIN
- SO: Windows 10
- Rede: Rede interna "rede-administrativa"
- MAC: 08:00:27:EF:FD:0F

**Verificação no cliente:**
```cmd
ipconfig
```

**Resultado obtido:**
- IPv4 Address: 172.16.1.16
- Subnet Mask: 255.255.255.0
- Default Gateway: 172.16.1.1

**Teste de persistência:**
Após reinicialização da CWIN, o IP 172.16.1.16 foi mantido, confirmando a reserva de endereço fixo.

### 4.3 Logs do Servidor

**Monitoramento de concessões:**
```bash
sudo tail -f /var/log/syslog | grep dhcp
```

**Leases ativos:**
```bash
sudo cat /var/lib/dhcp/dhcpd.leases
```

---

## 5. DESAFIOS E SOLUÇÕES

### 5.1 Problema: Mapeamento de teclado no VirtualBox

**Descrição:** Dificuldade para digitar caracteres especiais (como "/") na interface do VirtualBox com teclado Mac.

**Solução:** Configuração de Port Forwarding e acesso via SSH do Terminal do Mac, eliminando problemas de mapeamento de teclado.

### 5.2 Problema: Configuração inicial do Windows

**Descrição:** Processo de configuração inicial do Windows 10 demorado.

**Solução:** Utilização de Shift+F10 para acesso direto ao Prompt de Comando durante instalação, permitindo verificação imediata da configuração de rede.

---

## 6. EVIDÊNCIAS

### 6.1 Screenshots das Configurações

#### 6.1.1 Configuração das Interfaces de Rede no VirtualBox
<img width="600" alt="Configuração das 3 interfaces de rede" src="https://github.com/user-attachments/assets/2a474be1-101a-47ba-bcc7-44eec576d749" />

#### 6.1.2 Verificação dos IPs Configurados (`ip addr show`)

**Servidor Admin:**

<img width="500" alt="IP do servidor admin" src="https://github.com/user-attachments/assets/a9f8aba7-baa0-447a-9d40-c111f8673bd2" />

**Servidor Alunos:**

<img width="500" alt="IP do servidor alunos" src="https://github.com/user-attachments/assets/088cde18-25d4-44b7-b316-1c32ef1b5e37" />

#### 6.1.3 Arquivo de Configuração DHCP (`/etc/dhcp/dhcpd.conf`)
<img width="550" alt="Configuração do DHCP" src="https://github.com/user-attachments/assets/9f0f4e04-32c1-49ed-b0e5-809c8d487629" />

#### 6.1.4 Status do Serviço DHCP
<img width="550" alt="Status do serviço DHCP" src="https://github.com/user-attachments/assets/09f464a7-43f3-4f1a-8dca-d37f0b9a5371" />

#### 6.1.5 Verificação do IP Fixo - Cliente CWIN
<img width="500" alt="ipconfig CWIN" src="https://github.com/user-attachments/assets/d93b7102-e61c-45c2-8a0a-e21a0887087b" />



### 6.2 Arquivos de Configuração

- `/etc/netplan/50-cloud-init.yaml`
- `/etc/default/isc-dhcp-server`
- `/etc/dhcp/dhcpd.conf`

---

## 7. CONCLUSÃO

A implementação do servidor DHCP foi concluída com sucesso, atendendo todos os requisitos da atividade:

- Configuração de duas redes distintas com distribuição automática de IPs
- Reserva de endereço IP fixo para a estação CWIN
- Validação através de testes práticos
- Documentação completa do processo

O servidor está operacional e pronto para distribuir endereços IP automaticamente para clientes nas redes administrativa e de alunos, além de garantir que a máquina CWIN sempre receba o endereço 172.16.1.16.

---

## 8. REFERÊNCIAS

- Ubuntu Server Documentation: https://ubuntu.com/server/docs
- ISC DHCP Server Documentation: https://kb.isc.org/docs/isc-dhcp-44-manual-pages-dhcpdconf
- VirtualBox User Manual: https://www.virtualbox.org/manual/
- Netplan Reference: https://netplan.io/reference

---

## ANEXOS

### Anexo A - Comandos Úteis

**Reiniciar serviço DHCP:**
```bash
sudo systemctl restart isc-dhcp-server
```

**Ver logs em tempo real:**
```bash
sudo journalctl -u isc-dhcp-server -f
```

**Verificar sintaxe do dhcpd.conf:**
```bash
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

**Ver leases ativas:**
```bash
sudo dhcp-lease-list
```

### Anexo B - Troubleshooting

**Serviço não inicia:**
- Verificar sintaxe dos arquivos de configuração
- Confirmar que as interfaces especificadas existem
- Verificar logs: `sudo journalctl -xe`

**Cliente não recebe IP:**
- Verificar se a interface do servidor está UP
- Confirmar que o cliente está na rede correta
- Verificar firewall do servidor

**IP fixo não funciona:**
- Confirmar MAC address correto
- Verificar se a subnet do IP fixo está definida
- Reiniciar serviço DHCP após mudanças
