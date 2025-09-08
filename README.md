#AUTORES ORIGINAIS DO CÓDIGO

Lucas Demetrius Augusto e Fabio Henrique Cabrini

#AUTORES DO TRABALHO
Ronaldo Aparecido Monteiro Almeida RM:565017
Lucas Rowlands Abat RM: 562994
Lucas Flekner Branquinho RM: 562262
Vitor Bordalo Correa Guimarães RM: 561592

# README — Controle de LED (ESP32) via MQTT

> Projeto: Cliente MQTT para ESP32 que controla um LED (onboard) e publica valores de luminosidade.

---

## Visão geral

Este projeto contém um sketch para ESP32 que se conecta a uma rede Wi‑Fi, conecta-se a um broker MQTT, escuta comandos para ligar/desligar o LED onboard e publica periodicamente o estado do LED e o valor de luminosidade lido de um sensor (p.ex. LDR ou potenciômetro) no pino ADC 34.

O código usa as bibliotecas `WiFi.h` (incluída no core do ESP32) e `PubSubClient.h` (cliente MQTT).

---

## Funcionalidades principais

* Conexão com rede Wi‑Fi (SSID e senha configuráveis).
* Conexão com broker MQTT (endereço IP/porta configuráveis).
* Assinatura de tópico de comando para ligar/desligar o LED.
* Publicação periódica do estado do LED no tópico de atributos.
* Leitura do ADC (pino 34) e publicação do valor de luminosidade (0–100) em outro tópico MQTT.
* Rotinas de reconexão para Wi‑Fi e MQTT.
* Inicialização do LED com efeito de "blink" para indicar boot.

---

## Hardware necessário

* Placa ESP32 (ex.: ESP32 Dev Module).
* LED onboard (muitos módulos usam o GPIO2 / D4 como LED integrado).
* Sensor de luminosidade: LDR (fotorresistor) ou um potenciômetro para teste.
* Resistores e fios para formar um divisor de tensão com o LDR (ver seção Conexão).

---

## Mapeamento de pinos (padrão no código)

* `D4 / GPIO2` — LED (saída).
* `GPIO34` — Entrada analógica (sensor de luminosidade). Observe que GPIO34 é um pino ADC de leitura apenas (input-only) no ESP32.

> Se sua placa usar pinos diferentes, atualize a variável `default_D4` e `potPin` no código.

---

## Bibliotecas necessárias

* `WiFi.h` (core do ESP32)
* `PubSubClient.h` (instale pela Library Manager no Arduino IDE ou via PlatformIO)

---

## Configurações principais (no código)

No topo do sketch existem constantes editáveis:

```cpp
const char* default_SSID = "Wokwi-GUEST";
const char* default_PASSWORD = "";
const char* default_BROKER_MQTT = "20.163.23.245"; // IP do broker
const int default_BROKER_PORT = 1883;
const char* default_TOPICO_SUBSCRIBE = "/TEF/lamp001/cmd";
const char* default_TOPICO_PUBLISH_1 = "/TEF/lamp001/attrs";
const char* default_TOPICO_PUBLISH_2 = "/TEF/lamp001/attrs/l";
const char* default_ID_MQTT = "fiware_001";
const int default_D4 = 2; // pino do LED
const char* topicPrefix = "lamp001"; // prefixo usado para montar payloads de controle
```

Altere essas constantes conforme necessário antes de compilar e gravar na placa.

---

## Formato de mensagens MQTT

### Tópicos

* **Tópico de subscribe (comandos):** `default_TOPICO_SUBSCRIBE` (ex.: `/TEF/lamp001/cmd`).
* **Tópico de publish (estado):** `default_TOPICO_PUBLISH_1` (ex.: `/TEF/lamp001/attrs`). Envia `s|on` ou `s|off`.
* **Tópico de publish (luminosidade):** `default_TOPICO_PUBLISH_2` (ex.: `/TEF/lamp001/attrs/l`). Envia um valor numérico (0–100) como string.

### Payloads esperados

* Para ligar o LED: enviar **no tópico de comando** a string:

```
lamp001@on|
```

* Para desligar o LED: enviar **no tópico de comando** a string:

```
lamp001@off|
```

> O `topicPrefix` (no código `lamp001`) é usado como parte do payload, não do tópico. Assim o broker recebe comandos no tópico (ex.: `/TEF/lamp001/cmd`) e o payload é comparado com `topicPrefix + "@on|"` ou `topicPrefix + "@off|"`.

### Payloads publicados pelo dispositivo

* Estado do LED (em `default_TOPICO_PUBLISH_1`): `s|on` ou `s|off`.
* Luminosidade (em `default_TOPICO_PUBLISH_2`): um número 0–100 (como string), obtido pelo `map(sensorValue, 0, 4095, 0, 100)`.

---

## Exemplo de uso com mosquitto (línea de comando)

Para enviar comandos (substitua `BROKER` pelo IP do seu broker):

```bash
mosquitto_pub -h BROKER -t /TEF/lamp001/cmd -m "lamp001@on|"
mosquitto_pub -h BROKER -t /TEF/lamp001/cmd -m "lamp001@off|"
```

Para escutar os estados e a luminosidade:

```bash
mosquitto_sub -h BROKER -t "/TEF/lamp001/attrs" -v
mosquitto_sub -h BROKER -t "/TEF/lamp001/attrs/l" -v
```

---

## Como o código funciona (fluxo resumido)

1. `setup()` chama `InitOutput()` (pisca o LED) e inicializa Serial, Wi‑Fi e MQTT.
2. Após um `delay(5000)`, o sketch faz uma publicação inicial para `TOPICO_PUBLISH_1` com `"s|on"` (atente-se que essa publicação ocorre independentemente da conexão MQTT — ver sessões de melhorias).
3. `loop()` executa:

   * `VerificaConexoesWiFIEMQTT()` — tenta reconectar ao broker se necessário e garante conexão Wi‑Fi.
   * `EnviaEstadoOutputMQTT()` — publica o estado do LED (`s|on`/`s|off`) e imprime no Serial (há um `delay(1000)` no fim dessa função).
   * `handleLuminosity()` — lê ADC do pino 34, converte para 0–100 e publica em `TOPICO_PUBLISH_2`.
   * `MQTT.loop()` — processa mensagens MQTT e dispara `mqtt_callback` quando chegam comandos.
4. `mqtt_callback()` formata o payload recebido e compara com as strings esperadas (`lamp001@on|` / `lamp001@off|`) para acionar o LED e atualizar `EstadoSaida`.

---

## Observações importantes e possíveis problemas

* **Publicação inicial em `setup()`**: o `MQTT.publish(TOPICO_PUBLISH_1, "s|on")` em `setup()` é chamado antes de garantir que a conexão MQTT foi estabelecida. Em muitos casos essa mensagem será perdida. Recomenda‑se mover essa publicação para ser feita logo após a confirmação de conexão MQTT (por exemplo dentro do bloco `if (MQTT.connect(ID_MQTT))` em `reconnectMQTT()` ou checar `MQTT.connected()` antes de publicar).

* **Bloqueios (loops while/delay):** funções como `reconectWiFi()` e `reconnectMQTT()` usam loops bloqueantes (`while`/`delay`) que impedem o restante do código de rodar. Para aplicações mais sensíveis a tempo real, implemente reconexões não‑bloqueantes usando timers (millis) e estados.

* **Flood de publicações:** `EnviaEstadoOutputMQTT()` publica o estado em cada iteração (e possui `delay(1000)`), e `handleLuminosity()` também publica a cada loop. Isso gera mensagens MQTT a cada segundo; avalie se precisa reduzir a frequência ou publicar somente quando o valor mudar.

* **Uso de `const_cast<char*>` para variáveis de configuração:** as strings de configuração são declaradas como `const char*` e depois convertidas para `char*` com `const_cast`. Isso pode ser confuso e inseguro. Se pretende permitir edição em tempo de execução, use `String` ou buffers de char próprios; caso contrário mantenha `const char*` constantes.

* **Formato de payloads simples:** o projeto usa strings curtas com separadores (ex.: `s|on`). Para integração mais robusta com plataformas (ex.: Fiware, Home Assistant, etc.) recomenda‑se usar JSON (biblioteca `ArduinoJson`) para estruturar atributos.

* **Segurança:** credenciais Wi‑Fi estão em texto plano no sketch. Em projetos finais, tome cuidado com segurança (variáveis sensíveis fora do repositório, uso de TLS/MQTTs, autenticação no broker).

---

## Possíveis melhorias e extensões

* Publicar apenas mudanças de estado (reduz tráfego MQTT).
* Implementar QoS/retain conforme suporte do broker.
* Usar MQTT over TLS (MQTTS) para segurança e autenticação com usuário/senha.
* Reestruturar reconexões para serem não‑bloqueantes (usar `millis()`).
* Usar `ArduinoJson` para publicar mensagens em JSON com timestamps, IDs e metadados.
* Adicionar o suporte a OTA (over‑the‑air) para atualizar o firmware remotamente.
* Implementar watchdog e tratamento de exceções para maior robustez.

---

## Como modificar o comportamento (exemplos rápidos)

* **Alterar SSID/Senha:** edite `default_SSID` e `default_PASSWORD` no início do sketch.
* **Alterar broker MQTT:** mude `default_BROKER_MQTT` e `default_BROKER_PORT`.
* **Alterar tópicos:** ajuste as constantes `default_TOPICO_SUBSCRIBE`, `default_TOPICO_PUBLISH_1` e `default_TOPICO_PUBLISH_2`.
* **Alterar pino do LED:** troque `default_D4` (e, se necessário, verifique se o pino do LED da placa é diferente).

---

## Boas práticas para depuração

1. Abra o Monitor Serial (baud 115200) para acompanhar logs.
2. Verifique se o ESP32 obteve um IP (mensagem impressa no Serial).
3. Use `mosquitto_sub` para se inscrever nos tópicos e confirmar publicações.
4. Valide o payload de controle exatamente no formato `lamp001@on|`/`lamp001@off|` (caso contrário o callback não acionará).

---

## Licença

MIT — sinta‑se livre para usar e adaptar este código. Por favor, mantenha atribuição se for compartilhar publicamente.

---

## Contato / Agradecimentos

Se quiser, posso:

* Adaptar o sketch para usar JSON.
* Remover bloqueios e tornar reconexão não‑bloqueante.
* Converter payloads para um formato compatível com uma plataforma específica (ex.: Home Assistant).

Basta me dizer o que prefere que eu altere.
