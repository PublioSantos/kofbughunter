# kofbughunter

**[PT]** Ferramenta de apoio mecânico para a caça a bugs no Kof (JVM target).
Faz tudo que é repetitivo para que o esforço de IA fique só onde é inevitável:
gerar o repro e explicar a causa raiz.

**[EN]** Mechanical support tool for bug-hunting in Kof (JVM target).
Handles all repetitive work so AI effort is spent only where it cannot be replaced:
inventing the reproducer and explaining the root cause.

---

## Como funciona / How it works

**[PT]** O núcleo (`apoiador/main.kf`) é escrito em Kof — dogfooding deliberado.
Escrever o apoiador na própria linguagem investigada já rendeu bugs novos.
O wrapper shell (`kofbug`) só faz o que Kof não consegue: executar processos.

**[EN]** The core logic (`apoiador/main.kf`) is written in Kof itself — deliberate dogfooding.
Writing the support tool in the language under investigation has already surfaced new bugs.
The shell wrapper (`kofbug`) only handles what Kof cannot: spawning subprocesses.

---

## Divisão de trabalho / Division of work

| Etapa / Step | Responsável / Owner |
|---|---|
| Gerar o repro / Generate the reproducer | **IA / AI** |
| Compilar, executar, extrair bytecode / Build, run, disassemble | `kofbug` (shell) |
| Classificar a falha / Classify the fault | **Kof** |
| Comparar saída esperada × real / Diff expected vs actual | **Kof** |
| Deduplicar contra issues já reportadas / Dedup against filed issues | **Kof** |
| Preencher 5 das 8 seções do template / Fill 5 of 8 issue template sections | **Kof** |
| Description / Root Cause / Suggested Fix | **IA / AI** |
| Abrir a issue / File the issue | `gh` |

Das 8 seções obrigatórias do template, o Kof preenche 5. Sobram 3 — as que exigem análise real.  
*Of the 8 required template sections, Kof fills 5. The remaining 3 require real analysis.*

---

## Uso / Usage

```bash
cd ~/apoiador

./kofbug new meurepro           # scaffold repros/meurepro/main.kf + expected.txt
$EDITOR repros/meurepro/main.kf # write the reproducer and expected output
./kofbug run meurepro           # build + run + javap + classify + dedup
./kofbug focus meurepro Classe  # narrow bytecode to one class
./kofbug issue meurepro         # emit the pre-filled issue skeleton
```

**[PT]** Depois de abrir a issue, registre-a para que as próximas rodadas deduplicem:

**[EN]** After filing the issue, record it for future dedup:

```bash
./kofbug file RUNTIME:VerifyError:stack-underflow 403 "field named log corrupts method bodies"
```

Outros comandos / Other commands: `list`, `stats`, `known`, `build` (recompila o apoiador / recompiles the tool)

---

## Acompanhamento de versão / Version tracking

**[PT]** Todo comando que invoca o compilador consulta o feed de releases antes
(no máximo uma vez por dia) e **faz o upgrade automaticamente** se houver versão
nova. O toolchain ativo é o symlink `kof-toolchain/current`, então o upgrade é só
repontar o link. O tarball é verificado contra o `SHA256SUMS` da release antes de
ser extraído.

**[EN]** Every command that invokes the compiler checks the release feed first
(at most once a day) and **upgrades automatically** when a newer version exists.
The active toolchain is the `kof-toolchain/current` symlink, so an upgrade is
just a repointed link. The tarball is verified against the release `SHA256SUMS`
before extraction.

```bash
./kofbug version    # installed vs. latest upstream
./kofbug upgrade    # download + verify + install + rebuild
./kofbug regress    # re-run every repro; report what moved vs. baseline
./kofbug baseline   # freeze current verdicts as the new baseline
```

`KOFBUG_NO_UPDATE=1` fixa a versão atual / pins the current version.

**[PT]** Depois de um upgrade, `regress` é o que diz **quais bugs reportados foram
corrigidos**: cada repro guarda o veredito da primeira captura em `baseline.txt`, e
só o que mudou aparece (`FIXED`, `BROKE`, `CHANGED`). Um repro que sempre passou é
controle, não correção.

**[EN]** After an upgrade, `regress` is what tells you **which filed bugs got
fixed**: each repro stores its first verdict in `baseline.txt`, and only what moved
is reported (`FIXED`, `BROKE`, `CHANGED`). A repro that always passed is a control,
not a fix.

**[PT]** A linha `## Environment` das issues é medida do toolchain vivo, nunca fixa —
uma issue não pode alegar uma versão contra a qual o repro não rodou.

**[EN]** The `## Environment` line of every issue is probed from the live toolchain,
never hardcoded — an issue can never claim a version the repro was not run against.

---

## Vereditos / Verdicts

- **`NOVEL`** — nenhuma issue tem essa assinatura → candidato a bug novo  
  *no filed issue carries this signature → new bug candidate*
- **`LIKELY DUPLICATE`** — alguma issue compartilha a assinatura → confirmar mecanismo  
  *some filed issue shares the signature → confirm if the mechanism differs*
- **`NO FAULT DETECTED`** — compilou, rodou e bateu com o esperado  
  *compiled, ran, and matched expected output*

**[PT]** A assinatura é `KIND:CODE`, e para `VerifyError` inclui a razão do JVM
(`bad-return-type`, `stack-underflow`, …) — sem ela todos os erros de backend caem num balde só.

**[EN]** The signature is `KIND:CODE`, and for `VerifyError` also includes the JVM reason string
(`bad-return-type`, `stack-underflow`, …) — without it every backend fault collapses into one bucket.

No índice (`db/filed.txt`), uma entrada pode terminar em `:*` para correspondência por prefixo.  
*In the index (`db/filed.txt`), an entry may end in `:*` for prefix-match dedup.*

---

## Fonte da verdade / Source of truth

**[PT]** Antes de escrever qualquer repro, a forma se confere no **corpus estável**
do repositório Kof4j — `training/` + `learn/` + `docs/`. Uma forma que não está lá
não é Kof, e o compilador recusá-la não é bug.

**[EN]** Before writing any reproducer, the form is checked against the **stable
corpus** in the Kof4j repo — `training/` + `learn/` + `docs/`. A form that is not
there is not Kof, and the compiler rejecting it is not a bug.

| Documento / Document | O que responde / What it answers |
|---|---|
| `docs/language-reference/grammar.md` §5.3 | operadores que **não existem** / operators that do **not** exist |
| `docs/language-reference/lexical-structure.md` §1.1 | keywords exaustivas / exhaustive keyword list |
| `docs/bugs-and-gaps/known-bugs.md` | bugs já conhecidos + §"behaviors that LOOK like bugs but are expected" |
| `docs/bugs-and-gaps/specification-gaps.md` | SG-001–020: ausências **por design** / absences **by design** |
| `docs/development/` | em desenvolvimento / in development |
| `docs/development/future/` | só plano, zero código / plan only, zero code |

**[PT]** Duas regras que valem mais que qualquer heurística:

**[EN]** Two rules that outrank every heuristic:

1. **Se não for nativo do Kof, não usar.** Uma issue falsa não é só ruído de
   triagem — ela puxa o agente do Kof a *implementar* a API inventada.
   *If it is not native Kof, do not use it. A false issue does not just waste
   triage — it pulls the Kof agent toward implementing the invented API.*
2. **Caçar contra o branch `beta-0.4.0`, nunca contra a tag de release.** O branch
   está à frente; contra a tag, tudo que já foi corrigido reaparece como bug novo.
   *Hunt against the `beta-0.4.0` branch, never the release tag.*

```bash
./kofbug tip ~/Downloads/Kof4j   # compila o branch e fixa o toolchain nele
./kofbug lint meurepro           # recusa formas Kotlin/Java antes de virarem issue
```

`run` já faz o lint sozinho. O pin sobrevive ao auto-update: a versão do branch
é *menor* que a da tag mais nova, então sem o marcador o comparador faria
downgrade e ressuscitaria todos os bugs já corrigidos.
*`run` lints on its own. The pin survives auto-update: the branch version reads
lower than the newest tag, so without the marker the comparator would downgrade
and resurrect every already-fixed bug.*

---

## Bugs do Kof evitados no apoiador / Kof bugs deliberately avoided in the tool

**[PT]** O apoiador evita deliberadamente tudo que já se sabe quebrado — senão ele mesmo não compila.

**[EN]** The tool deliberately avoids everything known to be broken — otherwise it wouldn't compile itself.

- sem classes/interfaces genéricas próprias (#385) / no user-defined generic classes or interfaces
- sem campos de tipo-função (#388, #402) / no function-type fields
- sem primitivos anuláveis em campo ou parâmetro (#393, #398, #408) / no nullable primitive fields or parameters
- sem `List.reduce()` com retorno tipado (#394, #395) / no typed-return `List.reduce()`
- sem `String.charAt()` (#387) — usa `substring(i, i+1)` / uses `substring(i, i+1)` instead
- sem interpolação `${}` (#369) — concatenação explícita / explicit string concatenation
- sem campo chamado `log` (#403) — corrompe corpos de método / corrupts method bodies
- sem variável local `args` em `main()` (#397) / no local variable named `args` in `main()`
- `main(argv: String[])`, nunca `main(argv: List<String>)` — gera `main(ArrayList)` inutilizável  
  *`main(argv: List<String>)` compiles to `main(ArrayList)` which the JVM launcher rejects*
- valores de `kof.io` só com tipo inferido (`var f = File(p)`) — `val f: File` dá SEM011  
  *`kof.io` values only with inferred type — explicit annotation gives SEM011*
- `String.split()` devolve `String[]`, não `List<String>` (#372)
- sem `StringBuilder` — classe Java, não existe no Kof / `StringBuilder` is a Java class, not a Kof type
- sem `Math.abs()` — Kof tem `math.abs()` (namespace `kof.math`), não a classe Java / Kof has `math.abs()`, not the Java class
- sem `mutableListOf()` — função do Kotlin, não do Kof; use `listOf()` que já é mutável / Kotlin stdlib, not Kof; `listOf()` is already mutable
- sem `Pair<K,V>` embutido — não é stdlib do Kof; defina `record Pair(first: K, second: V)` / not Kof stdlib; define a record instead
- **nunca `val`** — palavra reservada para retornar erro no Kof; usar sempre `var` / **never `val`** — reserved word in Kof; always use `var`

---

## Fluxo atual / Current workflow

```
kofbughunter → encontra bug → ./kofbug run → ./kofbug issue → gh issue create
             → aplica fix   → ./kofbug pr <repro> <issue#> <kof4j-dir>
                            → PR aberto contra KofLang/Kof4j:beta-0.4.0
                            → Criadora da linguagem Kof revisa → fecha issue
```

`kofbug pr` compila o Kof4j localmente (Maven + JDK 25 do toolchain), verifica o repro contra o jar patched, e abre o PR diretamente.

---

## Resultados / Results

**[PT]** Usado para encontrar e registrar bugs reais no [KofLang/Kof4j](https://github.com/KofLang/Kof4j).
Issues abertas com este apoiador: #354–#427 (130 bugs em 12 rodadas).

*Projetado e construído como um pipeline semi-automatizado de caça a bugs de compilador/JVM, combinando repros gerados por IA, execução determinística, inspeção de bytecode, classificação de falhas, deduplicação por assinatura e scaffolding automatizado de issues.*

**[EN]** Used to find and file real bugs in [KofLang/Kof4j](https://github.com/KofLang/Kof4j).
Issues filed using this tool: #354–#427 (130 bugs across 12 rounds).
PRs with suggested fixes: [#428](https://github.com/KofLang/Kof4j/pull/428).

*Designed and built as a semi-automated compiler/JVM bug-hunting pipeline, combining AI-generated reproducers, deterministic execution, bytecode inspection, fault classification, signature-based deduplication and automated issue scaffolding.*
