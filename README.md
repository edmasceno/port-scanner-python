# 🛡️ Análise Técnica: Port Scanner TCP com Python (Sockets)

Este projeto consiste em uma ferramenta de reconhecimento de rede (Footprinting) que interage diretamente com a pilha TCP/IP do sistema operacional para identificar portas abertas em um host alvo.

## 🏗️ 1. Arquitetura do Socket
Ao utilizar a biblioteca `socket` do Python, o script solicita ao kernel a criação de um endpoint de comunicação.
* **`socket.AF_INET`**: Define a família de endereços como IPv4.
* **`socket.SOCK_STREAM`**: Especifica o protocolo de transporte **TCP** (Transmission Control Protocol). Diferente do UDP, o TCP é orientado à conexão, o que permite validar o estado da porta através do handshake.

## 🤝 2. Mecanismo de Detecção (TCP Three-Way Handshake)
O scanner funciona tentando completar o processo de abertura de conexão do protocolo TCP:
1.  **SYN**: O script envia um pacote de sincronização para a porta alvo.
2.  **Resposta do Alvo**:
    * **SYN-ACK**: A porta está **ABERTA** e pronta para receber conexões.
    * **RST (Reset)**: O host responde ativamente que a porta está **FECHADA**.
    * **Sem Resposta (Drop)**: Indica que a porta está **FILTRADA** por um firewall ou o host está offline.

## 🛠️ 3. Implementação com `connect_ex()`
O método `connect_ex()` é utilizado por ser mais eficiente em scripts de varredura. Em vez de lançar uma exceção que interromperia o fluxo do código, ele retorna um código de erro do sistema (errno):
* **Retorno 0**: Indica sucesso na operação. O handshake foi completado (Porta Aberta).
* **Retorno != 0**: Indica falha na conexão. Os códigos comuns são 111 (Connection refused no Linux) ou 10061 (no Windows).

## ⏱️ 4. Gerenciamento de Timeout
O uso de `socket.setdefaulttimeout()` é crítico para a performance da ferramenta. Sem a definição de um tempo limite (ex: 1 segundo), o script herdaria o timeout padrão do sistema operacional, que pode chegar a 30 segundos por porta. Isso tornaria a varredura inviável em ambientes com firewalls que simplesmente descartam pacotes sem responder (DROP).

## 🛡️ 5. Visão de Monitoramento (Blue Team)
A execução deste script gera telemetria que pode ser detectada por sistemas de monitoramento como Wazuh ou ferramentas de análise de tráfego:
* **Sysmon Event ID 3 (Network Connection)**: Registra múltiplas tentativas de conexão originadas do mesmo processo para portas sequenciais em um curto intervalo de tempo.
* **Detecção de SIEM**: O padrão de conexões rápidas e incompletas caracteriza uma atividade de "Reconnaissance" (Reconhecimento), permitindo a criação de alertas baseados em volume de tentativas de conexão por IP de origem.
