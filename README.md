# Using ANTLR with SourceAcademy Conductor

## starting point:
Refer to Sam's repository: https://github.com/tsammeow/conductor-runner-example

## define your grammar
create a file (say, grammar/SimpleLang.g4) with something like:
```antlr
grammar SimpleLang;

prog: expression EOF;

expression
    : expression op=('+'|'-') expression
    | expression op=('*'|'/') expression
    | INT
    | '(' expression ')'
    ;

INT: [0-9]+;
WS: [ \t\r\n]+ -> skip;
```

## generate parser & visitor
assuming you’ve got the antlr jar, run this (note the -visitor flag):

```bash
java -jar antlr-4.9.2-complete.jar -Dlanguage=JavaScript -visitor -o src/antlr grammar/SimpleLang.g4
```
this spits out your lexer, parser, and a visitor in src/antlr.

## implement your evaluator with a visitor
create a new file (e.g. src/SimpleLangEvaluator.ts) that uses the generated parser. for example:
```typescript
import antlr4 from 'antlr4';
import SimpleLangLexer from './antlr/SimpleLangLexer.js';
import SimpleLangParser from './antlr/SimpleLangParser.js';
import SimpleLangVisitor from './antlr/SimpleLangVisitor.js';
import { BasicEvaluator } from "conductor/dist/conductor/runner";
import { IRunnerPlugin } from "conductor/dist/conductor/runner/types";

// extend the generated visitor to evaluate expressions
class EvalVisitor extends SimpleLangVisitor {
  visitProg(ctx: any): number {
    return this.visit(ctx.expression());
  }

  visitExpression(ctx: any): number {
    if (ctx.INT()) {
      return parseInt(ctx.INT().getText(), 10);
    } else if (ctx.getChildCount() === 3) {
      const left = this.visit(ctx.getChild(0));
      const op = ctx.getChild(1).getText();
      const right = this.visit(ctx.getChild(2));
      switch (op) {
        case '+': return left + right;
        case '-': return left - right;
        case '*': return left * right;
        case '/': return left / right;
      }
    }
    return 0;
  }
}

export class SimpleLangEvaluator extends BasicEvaluator {
  async evaluateChunk(chunk: string): Promise<void> {
    // set up antlr's pipeline
    const chars = new antlr4.InputStream(chunk);
    const lexer = new SimpleLangLexer(chars);
    const tokens = new antlr4.CommonTokenStream(lexer);
    const parser = new SimpleLangParser(tokens);
    parser.buildParseTrees = true;
    const tree = parser.prog();

    // evaluate using our visitor
    const visitor = new EvalVisitor();
    const result = visitor.visit(tree);
    this.conductor.sendOutput(`result: ${result}`);
  }

  constructor(conductor: IRunnerPlugin) {
    super(conductor);
  }
}
```

## update your entry point
change src/index.ts to import your new evaluator:
```typescript
import { initialise } from "conductor/dist/conductor/runner/util/";
import { SimpleLangEvaluator } from "./SimpleLangEvaluator";

const { runnerPlugin, conduit } = initialise(SimpleLangEvaluator);
```

## bundle into a single js file
your rollup config (in rollup.config.js) already uses src/index.ts as entry, so just run:
```bash
yarn build
```
this produces a bundled file at dist/index.js that’s fully conductor-compatible.
