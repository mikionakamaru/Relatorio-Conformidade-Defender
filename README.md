# Relatório de Postura de Endpoint no Microsoft Defender

<img width="1080" height="1350" alt="image" src="https://github.com/user-attachments/assets/602f4c55-d135-4ff0-98ae-71aba91e9f26" />

Conjunto de queries KQL para o Advanced Hunting do Microsoft Defender XDR que
transforma dado do Defender em relatório que o cliente lê sem tradutor. Colunas
em português, status traduzidos, e cada linha já vem com uma ação recomendada.

A ideia veio de um problema chato: o portal do Defender mostra bastante coisa,
mas nada que você mande por e-mail pro gestor de TI do cliente e ele entenda
sozinho.

## Pra quem serve

Quem administra Defender em ambiente próprio ou em modelo MSSP e precisa
entregar relatório periódico de postura de endpoint.

## Duas famílias de query

O repositório tem duas pastas, e a diferença entre elas é o que precisa estar
licenciado no ambiente. Leia isto antes de rodar, evita a frustração de uma
query que "não funciona" quando na verdade é falta de licença.

### `conformidade-tvm/` — roda em qualquer ambiente com Vulnerability Management

Essas queries partem do `DeviceTvmSecureConfigurationAssessment`. Funcionam onde
o Microsoft Defender Vulnerability Management está ativo. É o requisito mais
comum e mais fácil de ter.

| Arquivo | Objetivo | Saída |
|---|---|---|
| `01_Relatorio_Detalhado.kql` | Máquina por máquina com situação, prioridade e ação recomendada | Tabela |
| `02_Resumo_Executivo.kql` | Contagem e percentual por situação | Tabela curta |
| `03_Grafico_SituacaoParque.kql` | Distribuição geral do parque | Pizza |
| `04_Grafico_DefasagemVacina.kql` | Máquinas por faixa de dias sem atualizar assinatura | Colunas |
| `05_Controles_NaoConformes.kql` | Controles de segurança falhando, ordenados por impacto | Tabela e colunas |
| `06_Hostnames_Duplicados.kql` | Máquinas com mais de um registro no Defender | Tabela |
| `07_Vulnerabilidades_ComExploitPublico.kql` | CVEs com exploit conhecido no parque | Tabela |

### `inventario-deviceinfo/` — precisa da tabela DeviceInfo

Essas queries partem do `DeviceInfo`, que traz o estado do sensor e o status de
onboarding de cada máquina. Nem todo licenciamento expõe essa tabela. Se a query
retornar "Failed to resolve table or column expression named 'DeviceInfo'", é
licença ou permissão, não erro da query. Rode `DeviceInfo | take 1` para testar.

| Arquivo | Objetivo | Saída |
|---|---|---|
| `01_Endpoints_Inativos.kql` | Máquinas com sensor inativo | Tabela |
| `02_Endpoints_Desatualizados.kql` | Máquinas com assinatura atrasada | Tabela |
| `03_Dispositivos_Can_Be_Onboarded.kql` | Máquinas conhecidas mas fora do Defender | Tabela |
| `04_Sensores_Ativos.kql` | Sensor ativo e reportando na janela | Tabela |
| `05_Sensores_Inativos.kql` | Sensor inativo, mal configurado ou silencioso | Tabela |
| `06_Baixa_Telemetria.kql` | Máquina onboardada que parou de mandar evento | Tabela |
| `07_Grafico_Sensores_Ativos_x_Inativos.kql` | Proporção de sensores por estado | Pizza |
| `08_Grafico_Onboarding_Status.kql` | Distribuição por status de onboarding | Colunas |
| `09_Grafico_Endpoints_Ativos_x_Inativos.kql` | Ativo x inativo, visão binária | Pizza |
| `10_Grafico_Endpoints_Desatualizados_x_Atualizados.kql` | Proporção de vacina em dia | Pizza |

As queries de `04` e `05` usam critério duplo: o sensor precisa estar Active
**e** ter reportado dentro da janela de dias definida no topo. Isso alinha o
resultado com a tela de asset do portal, que aplica a mesma régua de tempo.

## Como usar

1. Portal do Microsoft Defender, menu **Caça**, aba **Caça avançada**.
2. Cole o conteúdo do arquivo e execute.
3. Nas queries de gráfico, o resultado já vem renderizado.
4. Para exportar, use **Exportar** no canto superior direito da grade.

Os limites ficam nos `let` do topo de cada query. Ajuste conforme o SLA acordado
com o cliente.

```kusto
let Janela = 30d;
let AlertaVacina = 7;
let CriticoVacina = 14;
let DiasParaAtivo = 7;
```

## Requisitos

- Microsoft Defender for Endpoint implantado
- Microsoft Defender Vulnerability Management ativo (para a pasta `conformidade-tvm`)
- Tabela `DeviceInfo` acessível (para a pasta `inventario-deviceinfo`)
- Permissão de leitura no Advanced Hunting

## Limitações que você precisa conhecer antes de entregar

**Janela de 30 dias.** O Advanced Hunting guarda 30 dias. Máquina parada há mais
tempo desaparece do relatório inteiro, não aparece como problema. O número de
máquinas silenciosas é sempre um piso, nunca o total.

**As duas famílias contam universos ligeiramente diferentes.** `DeviceInfo` e
`DeviceTvmSecureConfigurationAssessment` são populados por caminhos diferentes,
então o total de máquinas de uma pasta pode não bater exato com o da outra. Não
é erro, são fontes distintas.

**"Sensor ativo" do DeviceInfo não é idêntico à tela de asset.** As duas fontes
calculam saúde do sensor em momentos diferentes. Para máquina na fronteira dos
dias definidos, uma fonte pode já ter virado a chave e a outra não. As queries
`04` e `05` reduzem isso com o critério duplo, mas a divergência de fronteira
não some por completo.

**O resultado é vivo, muda a cada dia.** O número de "dias sem reportar" é
calculado com `now()`, então uma máquina de fronteira pode trocar de lado
dependendo do dia da execução. Isso é o comportamento certo, não instabilidade.

**Hostname duplicado infla o resultado.** Máquina reinstalada sem limpar o
registro antigo aparece duas vezes, e o registro morto fica marcado como crítico
para sempre. Rode `conformidade-tvm/06_Hostnames_Duplicados.kql` antes de
entregar qualquer relatório.

**`render barchart` não existe.** O Advanced Hunting aceita `table`,
`columnchart`, `piechart`, `linechart`, `scatterchart`, `areachart`,
`stackedareachart` e `timechart`.

## Sobre os identificadores de configuração segura

A pasta `conformidade-tvm` usa dois identificadores:

- `scid-2010`: modo de operação do antivírus (ativo, passivo, somente EDR)
- `scid-2011`: versão da assinatura, versão do motor e data da última atualização

O campo `Context` vem como JSON aninhado e é lido por posição no array. Se a
Microsoft mudar o formato, é aqui que quebra primeiro.

## Como acompanhar evolução

O Advanced Hunting não guarda histórico além de 30 dias, então tendência não sai
da query. O caminho simples é exportar o CSV a cada rodada e versionar. Depois de
três ou quatro coletas você tem a curva que mostra se o trabalho surtiu efeito.

## Licença

MIT.
