# Comment-based tests messing with coverage

While reviewing and challenging the changes made to add ESM support to SonarJS ESLint Plugin, I noticed something suspect with the coverage data emitted by V8 when executing the unit test suite: some scripts were reported twice, with different content. After some investigation, I noticed that it only happened when comment-based tests were included into the executed test suite.

## Reproducer

The minimal way to reproduce the issue is to execute with `tsx` the comment-based test of rule S2004, alongside the unit test of rule S105, while requesting V8 to emit the coverage data, and inspect the coverage data files themselves.

```shell
rm coverage -rf && NODE_V8_COVERAGE=coverage tsx packages/jsts/src/rules/S105/unit.test.ts && NODE_V8_COVERAGE=coverage tsx packages/jsts/src/rules/S2004/cb.test.ts
```

The file `packages/jsts/src/rules/S105/rule.ts` is mentioned in two different coverage data files, which is unexpected since it is supposed to be covered _only_ by the test script `packages/jsts/src/rules/S105/unit.test.ts`. But, more concerning, it is mentioned three times in total.

The first mention is in the coverage data that was generated when V8 executed `packages/jsts/src/rules/S105/unit.test.ts`.

```json5
{
  "result": [
    // some scripts...
    {
      "scriptId":"186",
      "url":"file:///.../packages/jsts/src/rules/S105/rule.ts",
      "functions":[
        {
          "functionName":"",
          "ranges":[
            {
              "startOffset":0,
              "endOffset":3598,
              "count":1
            }
          ],
          "isBlockCoverage":true
        },
        // more functions
      ]
    }
    // more scripts
  ]
}
```

We can see that, according to this coverage data file, V8 executed a version of the script that consists of 3599 characters.

The second and third mentions are in the coverage data that was generated when V8 executed `packages/jsts/src/rules/S2004/cb.test.ts`.

```json5
{
  "result": [
    // some scripts
    {
      "scriptId": "915",
      "url": "file:///.../packages/jsts/src/rules/S105/rule.ts"
      "functions": [
        {
          "functionName": "",
          "ranges": [
            {
              "startOffset": 0,
              "endOffset": 3598,
              "count": 1
            }
          ],
          "isBlockCoverage": true
        }
        // more functions
      ]
    },
    // some scripts in-between
    {
      "scriptId": "5807",
      "url": "file:///.../packages/jsts/src/rules/S105/rule.ts",
      "functions": [
        {
          "functionName": "",
          "ranges": [
            {
              "startOffset": 0,
              "endOffset": 4441,
              "count": 1
            }
          ],
          "isBlockCoverage": true
        }
        // more functions
      ]
    }
    // more scripts
  ]
}
```

We can see that, according to this coverage data file, V8 executed two different flavors of the script:

* first, it executed a version of the script that consists of 3599 characters - which is consistent with the coverage data emitted when V8 executed `packages/jsts/src/rules/S105/unit.test.ts`
* then, it executed a version of the script that consists of 4442 characters.

## Consequence

The consequence of this messy coverage data is that it makes code coverage tool incapable to report coverage reliably.

This problem can easily be noticed by asking One Double Zero to generate a coverage report for `packages/jsts/src/rules/S105/rule.ts` after executing only its unit test script, and then after executing both its unit test script and S2004 comment-based one.

When executing only S105 unit test script, One Double Zero reports that `packages/jsts/src/rules/S105/rule.ts` is entirely covered.

```shell
rm coverage -rf && \
NODE_V8_COVERAGE=coverage tsx packages/jsts/src/rules/S105/unit.test.ts && \
odz report --coverage-directory=coverage --reporters=text --sources=packages/jsts/src/rules/S105/rule.ts
```
```shell
----------|---------|----------|---------|---------|-------------------
File      | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s 
----------|---------|----------|---------|---------|-------------------
All files |     100 |      100 |     100 |     100 |                   
 rule.ts  |     100 |      100 |     100 |     100 |                   
----------|---------|----------|---------|---------|-------------------
```

When executing S105 unit test script and, then, S2004 comment-based script, One Double Zero reports that `packages/jsts/src/rules/S105/rule.ts` is not covered at all.

```shell
rm coverage -rf && \
NODE_V8_COVERAGE=coverage tsx packages/jsts/src/rules/S105/unit.test.ts && \
NODE_V8_COVERAGE=coverage tsx packages/jsts/src/rules/S2004/cb.test.ts && \
odz report --coverage-directory=coverage --reporters=text --sources=packages/jsts/src/rules/S105/rule.ts
```
```shell
----------|---------|----------|---------|---------|-------------------
File      | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s 
----------|---------|----------|---------|---------|-------------------
All files |   14.28 |        0 |       0 |   14.28 |                   
 rule.ts  |   14.28 |        0 |       0 |   14.28 | 32-40             
----------|---------|----------|---------|---------|-------------------
```

When executing S2004 comment-based script and, then, S105 unit test script, One Double Zero reports that `packages/jsts/src/rules/S105/rule.ts` is partially covered.

```shell
rm coverage -rf && \
NODE_V8_COVERAGE=coverage tsx packages/jsts/src/rules/S2004/cb.test.ts && \
NODE_V8_COVERAGE=coverage tsx packages/jsts/src/rules/S105/unit.test.ts && \
odz report --coverage-directory=coverage --reporters=text --sources=packages/jsts/src/rules/S105/rule.ts
```
```shell
----------|---------|----------|---------|---------|-------------------
File      | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s 
----------|---------|----------|---------|---------|-------------------
All files |   85.71 |       50 |     100 |   85.71 |                   
 rule.ts  |   85.71 |       50 |     100 |   85.71 | 40                
----------|---------|----------|---------|---------|-------------------
```

To better highlight that those inconsistencies are induced by comment-based tests, let's execute S105 unit test script instead of its comment based one. In this case, regardless of the order of execution of the tests, the result is the same: One Double Zero reports that `packages/jsts/src/rules/S105/rule.ts` is entirely covered.

```shell
rm coverage -rf && \
NODE_V8_COVERAGE=coverage tsx packages/jsts/src/rules/S2004/unit.test.ts && \
NODE_V8_COVERAGE=coverage tsx packages/jsts/src/rules/S105/unit.test.ts && \
odz report --coverage-directory=coverage --reporters=text --sources=packages/jsts/src/rules/S105/rule.ts
```
```shell
rm coverage -rf && \
NODE_V8_COVERAGE=coverage tsx packages/jsts/src/rules/S105/unit.test.ts && \
NODE_V8_COVERAGE=coverage tsx packages/jsts/src/rules/S2004/unit.test.ts && \
odz report --coverage-directory=coverage --reporters=text --sources=packages/jsts/src/rules/S105/rule.ts
```
```shell
----------|---------|----------|---------|---------|-------------------
File      | % Stmts | % Branch | % Funcs | % Lines | Uncovered Line #s 
----------|---------|----------|---------|---------|-------------------
All files |     100 |      100 |     100 |     100 |                   
 rule.ts  |     100 |      100 |     100 |     100 |                   
----------|---------|----------|---------|---------|-------------------
```

## Investigation

There are two different issues that need to be investigated:

* the first one is the mention of `packages/jsts/src/rules/S105/rule.ts` in the coverage data generated when executing `packages/jsts/src/rules/S2004/cb.test.ts`;
* the second one is the fact that `packages/jsts/src/rules/S105/rule.ts` is reported there as executed twice, with two different contents.

### Issue #1

Comment-based tests are executed by the `check` function that is exported by the module `packages/jsts/tests/tools/testers/comment-based/checker.ts`. This module imports some other modules, and ultimately ends up importing the whole set of available rules. This explains why `packages/jsts/src/rules/S105/rule.ts` script ends up being executed by V8. Looking at the coverage data file, we can see that this is also the case for all the other rules modules.

```text
cb.test.ts
->
packages/jsts/tests/tools/testers/comment-based/checker.ts
```

### Issue #2

This one is tricky. The previously mentioned `check` function creates an instance of eslint `RuleTester`, configured with a custom parser module _path_: `packages/jsts/tests/tools/testers/comment-based/parser.ts`. This module imports some other modules, and ultimately ends up importing the whole set of available rules. By consequence, when the `RuleTester` instance import this `packages/jsts/tests/tools/testers/comment-based/parser.ts` module to retrieve the actual parser, it also ends up importing the whole set of available rules. But since ESLint `RuleTester` factory is exported by a CommonJS module, the imported flavors of the rule modules are the CommonJS ones.
