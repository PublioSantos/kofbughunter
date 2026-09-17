# kofbug — apoiador do bug hunt do Kof

Ferramenta de apoio para a caça a bugs no Kof 0.4.2-beta (JVM target).
Faz **tudo que é mecânico**, para que o esforço de IA fique só onde é
insubstituível: inventar o repro e explicar a causa raiz.

O núcleo (`apoiador/main.kf`) **é escrito em Kof** — dogfooding: escrever o
apoiador na própria linguagem investigada já rende bugs.

## Divisão de trabalho

| Etapa | Quem faz |
|---|---|
| Inventar o programa de repro | **IA** |
| Compilar, executar, extrair bytecode | `kofbug` (shell — Kof não tem API de processo) |
| Classificar a falha (SEM/PARSE/COMP002/VerifyError/…) | **Kof** |
| Comparar saída esperada × real, linha a linha | **Kof** |
| Deduplicar contra as issues já reportadas | **Kof** |
| Preencher Environment/Reproducer/Expected/Actual/Bytecode | **Kof** |
| Escrever Description / Root Cause / Suggested Fix | **IA** |
| Abrir a issue | `gh` |

Das 8 seções obrigatórias do template, o Kof preenche 5. Sobram 3 — as que
exigem análise de verdade.

## Uso

```bash
cd /home/publio/Downloads/install/kofbug

./kofbug new meurepro           # cria repros/meurepro/main.kf + expected.txt
$EDITOR repros/meurepro/main.kf # escreve o repro e a saída esperada
./kofbug run meurepro           # compila + executa + javap + classifica + dedup
./kofbug focus meurepro Classe  # reduz o bytecode capturado a uma classe
./kofbug issue meurepro         # imprime o esqueleto da issue já preenchido
```

Depois de abrir a issue, registre-a para que as próximas rodadas deduplicem
contra ela:

```bash
./kofbug file RUNTIME:VerifyError:stack-underflow 400 "field named log corrupts method bodies"
```

Outros comandos: `list`, `stats`, `known`, `build` (recompila o apoiador).

## Vereditos

- `NOVEL` — nenhuma issue reportada tem essa assinatura → candidato a bug novo
- `LIKELY DUPLICATE` — alguma issue compartilha a assinatura → confirmar se o
  mecanismo é diferente antes de reportar
- `NO FAULT DETECTED` — compilou, rodou e bateu com o esperado → não é bug

Assinatura = `KIND:CODE`, e para `VerifyError` também a razão do JVM
(`bad-return-type`, `stack-underflow`, `bad-operand-type`, …), porque sem ela
todas as falhas de backend caem num balde só.

No índice (`db/filed.txt`), uma entrada pode terminar em `:*` quando a razão
exata nunca foi registrada — vira correspondência por prefixo.

## Restrições do Kof respeitadas no apoiador

O apoiador evita deliberadamente tudo que já se sabe quebrado, senão ele
mesmo não compila:

- sem classes genéricas próprias, sem interfaces genéricas (#385)
- sem campos de tipo-função (#388)
- sem primitivos anuláveis em campo ou parâmetro (#393, #398)
- sem `List.reduce()` com retorno tipado (#394, #395)
- sem `String.charAt()` (#387) — usa `substring(i, i+1)`
- sem interpolação `${}` (#369) — concatenação explícita
- sem campo chamado `log` (corrompe corpos de método — bug desta sessão)
- sem variável local `args` em `main()` (#397)
- `main(argv: String[])`, nunca `main(argv: List<String>)` (gera
  `main(ArrayList)`, classe não executável)
- valores de `kof.io` só com tipo inferido (`var f = File(p)`) — a anotação
  explícita `val f: File` dá SEM011

`String.split()` devolve `String[]`, não `List<String>` (#372) — o apoiador
converte na mão em `readLines`.
