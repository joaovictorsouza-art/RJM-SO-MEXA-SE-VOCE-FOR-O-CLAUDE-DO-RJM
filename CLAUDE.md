# Controle de Notas e Pagamentos — instruções para o Claude

Planilha Google: **Controle de Notas e Pagamentos**
ID: `1aOqMSZpfCdyzDM5IfFoTa9r9nYDB6qs8NvFmUri-J3s`
Link: https://docs.google.com/spreadsheets/d/1aOqMSZpfCdyzDM5IfFoTa9r9nYDB6qs8NvFmUri-J3s/edit

Acesso pelos conectores Google Sheets / Google Drive (a rede do container não alcança docs.google.com).

## Regra número 1 — não alterar o que já existe

Os dados já cadastrados são importantes. **Nunca** editar, corrigir, padronizar ou apagar
lançamentos antigos, mesmo com erro aparente (fornecedor escrito de dois jeitos, soma
estranha, NF com dois nomes, "ATENÇAO" no valor, etc.). Se notar algo, apenas avise o usuário.

As únicas mudanças permitidas são:
1. **Inserir** linhas novas (nota fiscal recebida, boleto/documento novo).
2. **Mover** um documento de "A PAGAR" para "PAGOS <MÊS>" quando o usuário informar o pagamento.
3. Atualizar o cabeçalho do bloco afetado (texto "N lançamentos" e, se preciso, o intervalo do `=SUM`).

Nunca reordenar, reformatar, renomear abas, mudar agrupamentos existentes ou "melhorar" dados antigos.
Antes de qualquer escrita, **ler o trecho atual** da planilha (as linhas mudam de posição).
Fazer as inserções de forma atômica (um `update_spreadsheet` com todas as requisições) e,
depois, ler de volta para conferir.

## Estrutura

Linhas 1–2: título e instrução. Linha 3: vazia (3 linhas congeladas). Depois, blocos:

```
<TÍTULO DO BLOCO>   ...   "N lançamentos"   =SUM(<coluna de valor do bloco>)
<cabeçalho das colunas>
<linhas de dados>
<linha vazia>
```

Cada bloco (cabeçalho de colunas + dados) é um grupo de linhas (+/−). Inserir linhas **dentro**
do bloco (entre o cabeçalho de colunas e a última linha de dados) para que grupo e `SUM` se expandam.
Usar `insertDimension` com `inheritFromBefore: false` na posição da primeira linha de dados,
para herdar a formatação (data `dd/mm/yyyy`, moeda `"R$" #,##0.00`).
Gravar datas como datas e valores como números (não como texto).

### Aba `NOTAS FISCAIS` (colunas A–E)
`Data de recebimento | Nr da NF | NATUREZA | Fornecedor | Valor Total da NF`

- Blocos por mês: `OUTUBRO 2026`, `SETEMBRO 2026`… (mais recente no topo).
- Dentro do bloco, ordem por data de recebimento **decrescente** (mais nova em cima).
- NATUREZA usada: `VENDA`, `BONIFICAÇÃO`, `USO E CONSUMO` (seguir o CFOP/natureza da NF).
- Mês novo: criar bloco novo acima do mês anterior, no mesmo formato
  (título `MÊS AAAA`, "N lançamentos" na col. D, `=SUM` na col. E; linha vazia de separação; agrupar).

### Aba `DOCUMENTOS PARA PAGAMENTO` (colunas A–I)
`EMPRESA | FORNECEDOR | VENCIMENTO | PAGO EM | ENTIDADE | VALOR | NOTA FISCAL | DATA DE EMISSÃO | OBSERVAÇÃO`

- Primeiro bloco: **`A PAGAR`**, ordenado por VENCIMENTO **crescente**. "N lançamentos" na col. E, `=SUM` na col. F.
  Em "A PAGAR", ENTIDADE costuma ser `OK` (boleto em mãos), `BRADESCO CHEQUE`, ou vazio.
- Depois, blocos **`PAGOS <MÊS> <ANO>`** (mês do pagamento), mais recente no topo,
  ordenados por PAGO EM **decrescente**. Aqui ENTIDADE = **onde foi pago**
  (`LOTÉRICA`, `PIX`, `DINHEIRO`, `RJM PG`, …).
- EMPRESA: `JJN` ou `RIBEIRO` (até hoje). Definir pelo destinatário/CNPJ da NF; na dúvida, perguntar.

## Fluxo 1 — chegou nota fiscal (foto ou PDF)

1. Ler a NF: número, data de emissão, fornecedor, natureza/CFOP, valor total, destinatário, e as
   **duplicatas/parcelas** (vencimento e valor de cada uma). Ler também os boletos, se vierem.
2. Conferir se a NF já está lançada (buscar o número nas duas abas). Se já estiver, avisar e não duplicar.
3. Aba `NOTAS FISCAIS`: inserir 1 linha no bloco do mês de recebimento
   (data de recebimento = hoje, salvo se o usuário disser outra).
   Fornecedor: usar o nome **como já aparece** na planilha para esse fornecedor, se existir.
4. Aba `DOCUMENTOS PARA PAGAMENTO`, bloco `A PAGAR`: inserir **uma linha por parcela/boleto**,
   na posição certa pelo vencimento: EMPRESA, FORNECEDOR, VENCIMENTO, ENTIDADE (`OK` se o boleto
   veio junto; vazio se ainda não), VALOR da parcela, NOTA FISCAL, DATA DE EMISSÃO.
   Bonificação normalmente não gera boleto.
5. Atualizar os "N lançamentos" dos blocos tocados; conferir que os `=SUM` cobrem as linhas novas.
6. Responder com um resumo do que foi lançado (linhas, valores, vencimentos). Se algo na imagem
   estiver ilegível ou ambíguo, perguntar em vez de chutar.

## Fluxo 2 — documento foi pago

O usuário informa: qual documento (fornecedor/NF/valor/vencimento), **onde** foi pago e **quando**.

1. Localizar a linha em `A PAGAR`. Se houver mais de uma candidata, perguntar qual.
2. Copiar a linha **exatamente como está** (não corrigir nada), preenchendo:
   PAGO EM = data informada; ENTIDADE = onde foi pago (usar os termos já usados: `LOTÉRICA`, `PIX`, …).
3. Inserir no bloco `PAGOS <MÊS DO PAGAMENTO>` na posição por PAGO EM decrescente
   (criar o bloco acima dos demais "PAGOS" se o mês ainda não existir).
4. Apagar a linha original de `A PAGAR`.
5. Fazer 2–4 num único `update_spreadsheet`. Atualizar "N lançamentos" dos dois blocos e conferir os `=SUM`.
6. Responder com o que foi movido.
