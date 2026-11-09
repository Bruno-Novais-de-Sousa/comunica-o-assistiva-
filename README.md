# Bengala Assistiva Inteligente para Mobilidade Urbana de Pessoas com Deficiência Visual

Sistema embarcado desenvolvido para ampliar a **autonomia e a segurança na locomoção** de pessoas com deficiência visual. O dispositivo, acoplado a uma bengala/bastão, **mapeia o ambiente ao redor do usuário** e **previne colisões** por meio da integração de **sensores ultrassônicos de proximidade**, **feedback tátil (vibração)** e **alertas sonoros**.

<p align="center">
  <img src="img/prototipo_bengala.png" alt="Protótipo montado em bastão" width="360">
</p>

---

## Sumário

- [Visão geral](#visão-geral)
- [Funcionalidades](#funcionalidades)
- [Arquitetura do sistema](#arquitetura-do-sistema)
- [Componentes utilizados](#componentes-utilizados)
- [Esquema de ligação](#esquema-de-ligação)
- [Protótipo físico](#protótipo-físico)
- [Lógica de funcionamento](#lógica-de-funcionamento)
- [Estrutura do repositório](#estrutura-do-repositório)
- [Como executar](#como-executar)
- [Validação do protótipo](#validação-do-protótipo)
- [Limitações e melhorias futuras](#limitações-e-melhorias-futuras)
- [Minha contribuição](#minha-contribuição)
- [Tecnologias](#tecnologias)
- [Autor](#autor)
- [Licença](#licença)

---

## Visão geral

A locomoção em ambientes urbanos apresenta diversos obstáculos para pessoas com deficiência visual, como postes, lixeiras, veículos estacionados, desníveis e pessoas em circulação. A bengala tradicional detecta apenas o que o usuário alcança fisicamente com ela.

Este projeto propõe um **dispositivo embarcado complementar**, fixado ao bastão, capaz de:

1. Medir continuamente a distância até objetos próximos.
2. Interpretar essas medições em níveis de risco.
3. Comunicar o perigo ao usuário de forma **não visual**, por vibração e som, com intensidade proporcional à proximidade do obstáculo.

## Funcionalidades

- Detecção de obstáculos em tempo real por **dois sensores ultrassônicos HC-SR04**
- Uso da **menor distância** entre os dois sensores para decidir o nível de alerta
- **Filtro de leituras falsas** (valores fora da faixa de 2 a 300 cm são descartados)
- Feedback tátil por **dois motores de vibração**, acionados em caso de obstáculo muito próximo
- Alerta sonoro por **buzzer**, com cadência que acelera conforme o obstáculo se aproxima
- **Botão liga/desliga** no cabo do bastão
- Saída de **debug pela porta serial** (9600 baud) com a distância medida
- Alimentação portátil por bateria de 9 V

## Arquitetura do sistema

```
┌─────────────────┐   ┌─────────────────┐
│ HC-SR04 frontal │   │ HC-SR04 "B"     │
│   (D8 / D9)     │   │   (D10 / D11)   │
└────────┬────────┘   └────────┬────────┘
         └──────────┬──────────┘
                    ▼
        ┌───────────────────────┐        ┌──────────────┐
        │     Arduino Uno       │◄───────│ Botão (D2)   │
        │  Firmware C/C++       │        └──────────────┘
        │  • Leitura/filtragem  │
        │  • Menor distância    │
        │  • Níveis de alerta   │
        └───────────┬───────────┘
                    │
     ┌──────────────┼──────────────┐
     ▼              ▼              ▼
┌──────────┐  ┌───────────┐  ┌───────────┐
│ Buzzer   │  │ Transistor│  │ Transistor│
│  (D6)    │  │ NPN → M1  │  │ NPN → M2  │
│          │  │   (D4)    │  │   (D5)    │
└──────────┘  └───────────┘  └───────────┘

Alimentação: bateria 9 V
```

## Componentes utilizados

| Qtd. | Componente | Função |
|:----:|------------|--------|
| 1 | Arduino Uno | Unidade de processamento |
| 2 | Sensor ultrassônico HC-SR04 | Medição de distância |
| 2 | Motor de vibração (motor DC) | Feedback tátil |
| 2 | Transistor NPN | Acionamento dos motores (o Arduino não fornece corrente suficiente diretamente) |
| 2 | Resistores | Limitação de corrente na base dos transistores |
| 1 | Buzzer | Alerta sonoro |
| 1 | Botão (push button) | Liga/desliga do usuário |
| 1 | Capacitor | Estabilização/filtragem da alimentação |
| 1 | Bateria 9 V | Alimentação portátil |
| 1 | Protoboard e jumpers | Montagem do circuito |
| 1 | Bastão dobrável + suportes e velcro | Estrutura de fixação do dispositivo |

## Esquema de ligação

O circuito foi projetado e simulado antes da montagem física:

<p align="center">
</p>

| Dispositivo | Sinal | Pino do Arduino |
|-------------|-------|-----------------|
| Botão (com `INPUT_PULLUP`) | Entrada | `D2` |
| Sensor frontal | TRIG | `D8` |
| Sensor frontal | ECHO | `D9` |
| Sensor "B" | TRIG | `D10` |
| Sensor "B" | ECHO | `D11` |
| Motor de vibração 1 (via transistor) | Base | `D4` |
| Motor de vibração 2 (via transistor) | Base | `D5` |
| Buzzer | Sinal | `D6` |

## Protótipo físico

O circuito foi integrado a um **bastão dobrável**: um sensor fica na extremidade superior, outro na região central junto à caixa com a eletrônica, e o **botão de controle fica no cabo**, ao alcance da mão do usuário. Os componentes foram fixados com suportes impressos e fitas de velcro.

<p align="center">
</p>

## Lógica de funcionamento

O firmware (C/C++ para Arduino) executa o seguinte laço:

1. **Botão:** detecta a borda de descida (pino em `INPUT_PULLUP`) e alterna o estado `ligado`. Com o sistema desligado, motores e buzzer ficam apagados.
2. **Leitura:** dispara o pulso de 10 µs no TRIG e mede o tempo do ECHO com `pulseIn` (timeout de 30 ms). A distância em cm é `duração × 0,034 / 2`.
3. **Filtro:** leituras menores que 2 cm ou maiores que 300 cm (inclusive timeout) retornam `999`, ou seja, "sem obstáculo".
4. **Decisão:** usa a **menor** distância entre os dois sensores.
5. **Atuação:** aciona os atuadores conforme a faixa abaixo.

**Níveis de alerta:**

| Distância (menor entre os sensores) | Vibração | Buzzer |
|-------------------------------------|:--------:|--------|
| < 40 cm (muito perto) | Ligada (2 motores) | Contínuo |
| 40 a 99 cm (médio) | Desligada | Bipe 200 ms ligado / 300 ms desligado |
| 100 a 149 cm (longe) | Desligada | Bipe 300 ms ligado / 700 ms desligado |
| ≥ 150 cm | Desligada | Desligado |

**Trecho principal do firmware:**

```cpp
long distF = medirDistancia(TRIG_F, ECHO_F);
long distB = medirDistancia(TRIG_B, ECHO_B);
long menorDistancia = min(distF, distB);

if (menorDistancia < 40) {
  digitalWrite(MOTOR_4, HIGH);
  digitalWrite(MOTOR_5, HIGH);
  digitalWrite(BUZZER, HIGH);
}
else if (menorDistancia < 100) {
  // bipe rápido, sem vibração
}
else if (menorDistancia < 150) {
  // bipe lento, sem vibração
}
else {
  // tudo desligado
}
```

O código completo está em [`src/bengala_assistiva/bengala_assistiva.ino`](src/bengala_assistiva/bengala_assistiva.ino).

## Estrutura do repositório

```
.
├── src/
│   └── bengala_assistiva/
│       └── bengala_assistiva.ino   # Firmware (Arduino C/C++)
├── docs/
│   └── relatorio.pdf
├── img/
│   ├── esquema_circuito.png        # Esquema elétrico
│   └── prototipo_bengala.png       # Foto do protótipo
└── README.md
```

## Como executar

**Pré-requisitos**

- Arduino Uno
- [Arduino IDE](https://www.arduino.cc/en/software)
- Circuito montado conforme o [esquema de ligação](#esquema-de-ligação)

**Passos**

1. Clone o repositório:
   ```bash
   git clone https://github.com/seu-usuario/seu-repositorio.git
   ```
2. Abra `src/bengala_assistiva/bengala_assistiva.ino` na Arduino IDE.
3. Selecione **Arduino Uno** e a porta correta em **Ferramentas**.
4. Clique em **Carregar** (Upload).
5. Para acompanhar as distâncias, abra o **Monitor Serial** em 9600 baud.
6. Desconecte o USB, alimente com a bateria de 9 V e use o botão no cabo para ligar/desligar.

## Validação do protótipo

O protótipo funcional foi validado com testes que verificam:

- **Precisão das medições:** comparação das distâncias lidas (Monitor Serial) com medidas reais.
- **Faixas de alerta:** conferência dos limiares de 40, 100 e 150 cm.
- **Obstáculos variados:** paredes, objetos pequenos, pessoas e superfícies de materiais diferentes.
- **Coerência do feedback:** vibração e som condizentes com a proximidade.
- **Autonomia da bateria:** tempo de operação contínua.

> Inclua aqui tabelas de resultados, gráficos, fotos ou vídeos dos testes.

## Limitações e melhorias futuras

- O uso de `delay()` nos bipes bloqueia o programa por até 1 s, o que atrasa a leitura do botão e das distâncias durante o alerta. Uma melhoria é usar `millis()` para temporização sem bloqueio.
- O botão não possui tratamento de debounce por software.
- A vibração só atua na faixa crítica (< 40 cm); níveis intermediários de vibração podem ser explorados com PWM.
- Sensores ultrassônicos têm cone de detecção limitado e podem falhar com superfícies que absorvem ou desviam o som.
- Bateria de 9 V tem autonomia reduzida com motores e buzzer ativos.
- Possíveis evoluções: sensores ToF/LiDAR, alertas por voz, integração com GPS, bateria recarregável e PCB dedicada, testes de usabilidade com usuários reais.

## Minha contribuição

- Desenvolvimento da **lógica de programação** do firmware
- Definição da **arquitetura do circuito** eletrônico
- **Validação do protótipo funcional**, incluindo testes e calibração

## Tecnologias

- **Arduino Uno** e **C/C++**
- Sensores ultrassônicos HC-SR04
- Motores de vibração, buzzer e transistores NPN

## Autor

- **Nome do Autor** – [GitHub](https://github.com/seu-usuario)
- Instituição / Curso / Disciplina: _preencher_

## Licença

Projeto de caráter acadêmico. Defina aqui a licença desejada (por exemplo, [MIT](https://opensource.org/licenses/MIT)).
