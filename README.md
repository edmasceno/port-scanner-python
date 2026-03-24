# 🛡️ Análise Técnica: Port Scanner com Python (Sockets)

Este projeto documenta a criação de uma ferramenta de reconhecimento de rede (Footprinting) que opera diretamente na camada de transporte do modelo OSI. O objetivo foi entender a interação entre o código Python e a pilha TCP/IP do kernel.

---

### 🔍 1. A Biblioteca Socket e a Interface do Kernel
O comando `socket.socket(socket.AF_INET, socket.SOCK_STREAM)` não apenas cria um objeto em Python, ele solicita ao sistema operacional a abertura de um descritor de arquivo de rede.

* **AF_INET (Address Family):** Especifica que o scanner utilizará endereços **IPv4**. Se o objetivo fosse escanear redes IPv6, seria necessário utilizar `AF_INET6`.
* **SOCK_STREAM:** Define que o tipo de socket é orientado a fluxo (Stream), o que invoca o protocolo **TCP**. O TCP é escolhido para varreduras de precisão porque exige a confirmação da conexão, ao contrário do `SOCK_DGRAM` (UDP), que é "fire and forget".

---

### 🤝 2. O Mecanismo de Conexão (TCP Three-Way Handshake)
O script funciona tentando completar o aperto de mão de três vias. O estado da porta é determinado pela resposta do kernel do alvo:

1.  **SYN (Synchronize):** O script envia um pacote com a flag SYN ativa para o IP e porta alvo.
2.  **Resposta do Alvo:**
    * **SYN-ACK:** A porta está **ABERTA** e pronta para receber conexões.
    * **RST (Reset):** O alvo recebeu o pedido, mas a porta está **FECHADA**. O kernel do alvo encerra a tentativa imediatamente.
    * **Sem Resposta (Timeout):** O pacote foi descartado (**DROP**) ou rejeitado por um firewall. A porta é considerada **FILTRADA**.

---

### 💻 3. Lógica de Programação: `connect_ex()` vs `connect()`
A escolha do método `connect_ex()` é puramente técnica e voltada para a eficiência:

* **`connect()`:** Lança uma exceção (erro) se a porta estiver fechada. Isso exigiria um bloco `try/except` para cada porta, tornando o código mais lento e verboso.
* **`connect_ex()`:** Retorna um código de erro numérico direto do sistema (C-style error code).
    * **Retorno 0:** A operação foi bem-sucedida (Porta Aberta).
    * **Retorno 111 (Linux) / 10061 (Windows):** Conexão recusada (Porta Fechada).

---

### ⏱️ 4. Gestão de Timeouts e Comportamento de Firewall
O uso de `socket.setdefaulttimeout()` é a única forma de evitar que o scanner fique "preso" em portas filtradas. 

Quando um firewall está presente, ele não responde com um pacote RST (que fecharia a conexão na hora); ele simplesmente ignora o pacote SYN. Sem um timeout definido, o script esperaria o tempo padrão do sistema (que pode chegar a 30-60 segundos por porta), inviabilizando a ferramenta.

---

### 🛡️ 5. Visão de SOC (Detecção e Telemetria)
Embora o script seja simples, o rastro digital gerado em um ambiente monitorado como o **Wazuh** é evidente:

* **Sysmon Event ID 3 (Network Connection):** Cada tentativa de conexão gera um log. Um analista SOC identificará o ataque ao notar centenas de eventos ID 3 originados do mesmo `ProcessID` (seu script python) destinados a portas diferentes em um intervalo curto de tempo.
* **Detecção de Varredura:** O SIEM utiliza contadores (thresholds). Se um único IP de origem gera mais de X conexões TCP falhas em Y segundos, o alerta de **Port Sweeping** ou **Reconnaissance** é disparado automaticamente.

---

### 📝 Resumo de Aprendizado Técnico
* **Camada OSI:** O script opera na Camada 4 (Transporte).
* **Primitivas de Rede:** Uso de `AF_INET` e `SOCK_STREAM` para manipulação de bits na rede.
* **Tratamento de Erros:** Utilização de códigos de retorno do kernel para determinar estados de porta.
