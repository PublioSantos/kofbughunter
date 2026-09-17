# kofbughunter

**[PT]** Ferramenta de apoio mecânico para a caça a bugs no Kof 0.4.2-beta (JVM target).
Faz tudo que é repetitivo para que o esforço de IA fique só onde é inevitável:
gerar o repro e explicar a causa raiz.

**[EN]** Mechanical support tool for bug-hunting in Kof 0.4.2-beta (JVM target).
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
| Inventar o repro / Invent the reproducer | **IA / AI** |
| Compilar, executar, extrair bytecode / Build, run, disassemble | `kofbug` (shell) |
| Classificar a falha / Classify the fault | **Kof** |
| Comparar saída esperada × real / Diff expected vs actual | **Kof** |
| Deduplicar contra issues já reportadas / Dedup against filed issues | **Kof** |
| Preencher 5 das 8 seções do template / Fill 5 of 8 issue template sections | **Kof** |
| Description / Root Cause / Suggested Fix | **IA / AI** |
| Abrir a issue / File the issue | `gh` |

Das 8 seções obrigatórias do template, o Kof preenche 5. Sobram 3 — as que exigem análise.  
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

---

## Resultados / Results

**[PT]** Usado para encontrar e registrar bugs reais no [KofLang/Kof4j](https://github.com/KofLang/Kof4j).
Issues abertas com este apoiador: #354–#409 (110 bugs em 10 rodadas).

**[EN]** Used to find and file real bugs in [KofLang/Kof4j](https://github.com/KofLang/Kof4j).
Issues filed using this tool: #354–#409 (110 bugs across 10 rounds).
