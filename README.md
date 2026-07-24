# Relatório de Conformidade do Microsoft Defender

<img width="1080" height="1350" alt="image" src="https://github.com/user-attachments/assets/602f4c55-d135-4ff0-98ae-71aba91e9f26" />

Conjunto de queries KQL para o Advanced Hunting do Microsoft Defender XDR que
transforma dado de avaliação de configuração em relatório que o cliente lê sem
tradutor.

A ideia veio de um problema chato: o portal do Defender mostra bastante coisa,
mas nada que você mande por e-mail pro gestor de TI do cliente e ele entenda
sozinho. Aqui as colunas têm nome em português, os status vêm traduzidos e cada
linha já vem com uma ação recomendada.

## Pra quem serve

Quem administra Defender em ambiente próprio ou em modelo MSSP e precisa
entregar relatório periódico de postura de endpoint.

## O que cada query faz

| Arquivo | Objetivo | Saída |
|---|---|---|
| `01_Relatorio_Detalhado.kql` | Lista máquina por máquina com situação, prioridade e ação recomendada | Tabela para exportar em CSV |
| `02_Resumo_Executivo.kql` | Contagem e percentual por situação | Tabela curta para abrir reunião |
| `03_Grafico_SituacaoParque.kql` | Distribuição geral do parque | Gráfico de pizza |
| `04_Grafico_DefasagemVacina.kql` | Máquinas por faixa de dias sem atualizar assinatura | Gráfico de colunas |
| `05_Controles_NaoConformes.kql` | Controles de segurança falhando no ambiente, ordenados por impacto | Tabela e gráfico de colunas |
| `06_Hostnames_Duplicados.kql` | Máquinas com mais de um registro no Defender | Tabela |

## Como usar

1. Portal do Microsoft Defender, menu **Caça** (Hunting), aba **Caça avançada**.
2. Cole o conteúdo do arquivo e execute.
3. Para as queries de gráfico, o resultado já vem renderizado.
4. Para exportar, use **Exportar** no canto superior direito da grade.

Os limites ficam nos `let` do topo de cada query:

```kusto
let Janela = 30d;
let AlertaVacina = 7;
let CriticoVacina = 14;
let AlertaComunicacao = 7;
```

Ajuste conforme o SLA acordado com o cliente.

## Requisitos

- Microsoft Defender for Endpoint implantado
- Microsoft Defender Vulnerability Management ativo (as tabelas `DeviceTvm*` dependem dele)
- Permissão de leitura no Advanced Hunting

## Limitações que você precisa conhecer antes de entregar

**Janela de 30 dias.** O Advanced Hunting guarda 30 dias. Máquina que parou de
reportar há mais tempo desaparece do relatório inteiro, não aparece como
problema. O número de máquinas silenciosas é sempre um piso, nunca o total.

**Não detecta máquina sem o Defender.** Essas queries partem das tabelas de
avaliação, então só enxergam quem já está "onboardado". Máquina com status
"Can be onboarded" não gera avaliação e não aparece aqui. Para isso é preciso a
tabela `DeviceInfo`, a API do MDE (`/api/machines`) ou a exportação do
inventário de dispositivos no portal.

**"Dias sem reportar" não é o mesmo que "Última visualização" do portal.** A
coluna mede quando saiu a última avaliação de configuração, não o último sinal
do sensor. As duas cadências são diferentes e os números não batem. Se o cliente
comparar, explique antes que ele pergunte.

**Hostname duplicado infla o resultado.** Máquina reinstalada sem limpar o
registro antigo aparece duas vezes, e o registro morto fica marcado como crítico
para sempre. Rode a query 06 antes de entregar qualquer relatório.

**`render barchart` não existe.** O Advanced Hunting aceita `table`,
`columnchart`, `piechart`, `linechart`, `scatterchart`, `areachart`,
`stackedareachart` e `timechart`.

## Sobre os identificadores usados

As queries usam dois identificadores de configuração segura:

- `scid-2010`: modo de operação do antivírus (ativo, passivo, somente EDR)
- `scid-2011`: versão da assinatura, versão do motor e data da última atualização

O campo `Context` vem como JSON aninhado e é lido por posição no array. Se a
Microsoft mudar o formato, é aqui que quebra primeiro.

## Como acompanhar evolução

O Advanced Hunting não guarda histórico além de 30 dias, então tendência não sai
da query. O caminho simples é exportar o CSV da query 02 a cada rodada e
versionar. Depois de três ou quatro coletas você tem a curva que mostra se o
trabalho está surtindo efeito.

## Licença

MIT.
