
# 🛡️ Home SOC Lab — Detecção de Brute-Force SSH com Wazuh

Laboratório doméstico de segurança, isolado em rede virtual, construído para simular um cenário real de ataque e detecção, usando Wazuh como plataforma de SIEM/XDR open-source.

## 📌 Objetivo

Montar um ambiente de SOC (Security Operations Center) funcional, isolado da rede de produção, capaz de:

- Simular ataques reais contra um host Windows
- Detectar e correlacionar esses ataques através de um SIEM
- Documentar o processo de troubleshooting real enfrentado durante a construção do ambiente

Este projeto faz parte da minha transição de carreira de Analista de Suporte de TI (Infraestrutura & Cloud) para Cibersegurança, servindo como evidência prática de habilidades de SOC, hardening e resposta a incidentes.

## 🏗️ Arquitetura

```
┌─────────────────────────────────────────────────────────┐
│              VirtualBox - Internal Network                │
│                    "labsec" (192.168.56.0/24)              │
│                                                             │
│   ┌──────────────┐      ┌──────────────┐     ┌──────────┐│
│   │ Kali Linux   │─────▶│ Wazuh Manager│◀────│ Windows  ││
│   │ (Atacante)   │ ssh  │   + Dashboard │wazuh│  Alvo    ││
│   │ 192.168.56.20│brute │ 192.168.56.10 │agent│192.168.56│
│   │              │force │               │     │   .30    ││
│   └──────────────┘      └──────────────┘     └──────────┘│
│         │                                           │      │
└─────────┼───────────────────────────────────────────┼──────┘
          │ (NAT - saída p/ internet, updates)         │
          ▼                                            ▼
      Internet                          (Isolado - sem NAT durante o ataque)
```

**Tipo de rede:** VirtualBox Internal Network — escolhida deliberadamente por não criar nenhuma interface no host nem permitir roteamento externo, garantindo isolamento total entre o laboratório e a rede doméstica real.

## 🧰 Stack utilizada

| Componente | Função | IP |
| --- | --- | --- |
| Kali Linux 2026.1 | Máquina atacante (nmap, hydra) | 192.168.56.20 |
| Wazuh 4.14 (all-in-one) | SIEM — manager + indexer + dashboard | 192.168.56.10 |
| Windows 11 | Host alvo, com OpenSSH Server + Wazuh Agent | 192.168.56.30 |

## ⚙️ Setup do ambiente

### Rede

- Criada uma **Internal Network** dedicada no VirtualBox (`labsec`), isolando as três VMs entre si e do restante da rede doméstica
- IPs estáticos definidos manualmente (sem DHCP) em cada VM:
    - Kali: `netplan`/`nmtui` → `192.168.56.20/24`
    - Wazuh manager (Ubuntu Server): `netplan` → `192.168.56.10/24`
    - Windows alvo: `New-NetIPAddress` (PowerShell) → `192.168.56.30/24`
- Um segundo adaptador NAT foi usado **temporariamente** em cada VM sempre que era necessário baixar pacotes/atualizações, sendo removido em seguida para preservar o isolamento do ambiente (prática de minimizar superfície de exposição)

### Wazuh Manager

Instalado via script oficial all-in-one:

```bash
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

Dashboard acessado remotamente através de **túnel SSH** a partir do Kali (já que a porta 443 do manager não é exposta fora da rede isolada):

```bash
ssh -L 8443:localhost:443 ubuntu@192.168.56.10
# Acesso via navegador do Kali: https://localhost:8443
```

### Agente no host Windows

1. Habilitado o **OpenSSH Server** nativo do Windows (alternativa ao RDP, que não é suportado na edição Home):

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

1. Agente Wazuh instalado via comando gerado pelo próprio dashboard (seção **Agents → Deploy new agent**)
2. Configuração customizada em `ossec.conf` para monitorar o canal de eventos do OpenSSH, que **não é monitorado por padrão** pelo agente:

```xml
<localfile>
  <location>OpenSSH/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

## 🎯 Simulação do ataque

**Reconhecimento (recon):**

```bash
nmap -sV -O 192.168.56.30          # varredura inicial de serviços/SO
nmap -sS -p- 192.168.56.30         # varredura completa das 65535 portas
nmap -sV -p 22 192.168.56.30       # confirmação do serviço SSH
```

**Ataque de força bruta (brute-force) via SSH:**

```bash
hydra -t 1 -w 5 -V -L usuario.txt -P senhas_teste.txt ssh://192.168.56.30
```

Resultado: a senha correta foi identificada na 9ª tentativa, após 8 falhas sequenciais — padrão clássico de ataque de dicionário.

```
[22][ssh] host: 192.168.56.30   login: windows   password: windows
1 of 1 target successfully completed, 1 valid password found
```

## 📊 Detecção no Wazuh

O dashboard confirmou a detecção completa do ataque:

| Métrica | Valor |
| --- | --- |
| Total de eventos no período | 450 |
| Falhas de autenticação | 90 |
| Autenticações bem-sucedidas | 39 |
| Alertas de nível crítico (12+) | 0 |

Regras acionadas incluíram `Logon Failure - Unknown` (múltiplas tentativas falhas) e `Windows Logon Success` (momento da invasão bem-sucedida), com picos claramente visíveis na linha do tempo, coincidindo com a janela de execução do `hydra`.

> 
> 
> 
> <img width="975" height="433" alt="image" src="https://github.com/user-attachments/assets/f56674a6-090b-4831-8471-902426f0e1e8" />
> 

## 🔧 Troubleshooting — Problemas reais enfrentados e resolvidos

Esta seção documenta os obstáculos técnicos reais do processo — considerado o conteúdo mais valioso do laboratório, pois demonstra capacidade de diagnóstico, não apenas execução de tutorial.

### 1. Erro de sincronização de tempo no Kali

**Sintoma:** `ping` retornava avisos de `time of day goes back`, com respostas duplicadas e corrompidas.
**Causa:** relógio interno da VM dessincronizado após pausa/retomada.
**Solução:**

```bash
sudo timedatectl set-ntp true
sudo hwclock --systohc
sudo reboot
```

### 2. Interface de rede sem IP (DHCP inexistente)

**Sintoma:** `eth0` do Kali ficava “tentando conexão” indefinidamente.
**Causa:** redes do tipo Internal Network não possuem servidor DHCP.
**Solução:** IP estático configurado via `nmtui`.

### 3. Firewall do Windows bloqueando ping de entrada

**Sintoma:** Windows conseguia pingar o Wazuh manager, mas o inverso falhava.
**Causa:** comportamento assimétrico padrão do Windows Firewall (bloqueia ICMP de entrada por padrão).
**Solução:** regra específica de ICMP, em vez de desativar o firewall inteiro:

```powershell
netsh advfirewall firewall add rule name="Allow ICMPv4-In" protocol=icmpv4:8,any dir=in action=allow
```

### 4. RDP indisponível na edição Windows Home

**Causa:** Área de Trabalho Remota como servidor só existe em edições Pro/Enterprise/Education.
**Solução:** pivô para OpenSSH Server, nativo em qualquer edição do Windows, mantendo validade equivalente do caso de uso (mesma lógica de detecção de brute-force).

### 5. Porta 22 aparecendo como “filtered” mesmo com o serviço ativo

**Sintoma:** `netstat` confirmava o SSH escutando localmente, mas o nmap externo via a porta como filtrada.
**Causa raiz:** o Windows classifica redes desconhecidas automaticamente como perfil **“Public”**, e a regra de firewall criada pelo instalador do OpenSSH só liberava o perfil **“Private”**.
**Solução:**

```powershell
Get-NetFirewallRule -DisplayName "OpenSSH SSH Server (sshd)" | Set-NetFirewallRule -Profile Any
```

> Esse comportamento é, na verdade, um exemplo de *hardening* funcionando como esperado — o Windows por padrão restringe mais em redes não confiáveis.
> 

### 6. Hydra falhando com “too many connection errors”

**Sintoma:** ataques de força bruta paralelos (`-t 14` padrão) falhavam completamente.
**Causa:** o serviço `sshd` ficou num estado instável acumulando conexões não finalizadas das tentativas anteriores.
**Solução:** redução de paralelismo e adição de intervalo entre tentativas, além de reiniciar o serviço:

```powershell
Restart-Service sshd
```

```bash
hydra -t 1 -w 5 -V -L usuario.txt -P senhas_teste.txt ssh://192.168.56.30
```

### 7. Agente Wazuh não reportava eventos de autenticação SSH

**Sintoma:** mesmo com o ataque ocorrendo normalmente, o dashboard mostrava `Authentication failure: 0`.
**Causa raiz:** o agente Wazuh no Windows, por padrão, monitora apenas os canais `Application`, `Security` e `System` do Visualizador de Eventos — o OpenSSH grava seus logs em um canal separado (`OpenSSH/Operational`), não incluído por padrão.
**Solução:** adição manual do canal customizado no `ossec.conf` do agente, seguida de reinício do serviço:

```powershell
Restart-Service WazuhSvc
```

> Esse foi o problema mais sutil do projeto — reforça a importância de entender exatamente **quais fontes de log** um SIEM está (ou não está) ingerindo, em vez de assumir cobertura total por padrão.
> 

## ✅ Resultados e aprendizados

- Montagem completa de uma rede isolada do zero, sem exposição a produção
- Deploy funcional de um SIEM open-source (Wazuh) em arquitetura all-in-one
- Simulação de um ataque real (brute-force SSH) e validação completa da cadeia de detecção, do evento bruto até o dashboard
- Troubleshooting de 7 problemas técnicos distintos, envolvendo rede, firewall, sincronização de tempo e configuração de SIEM
- Prática direta de conceitos de hardening (perfis de firewall, princípio de isolamento de rede, regras específicas vs. desativação total)
