---
name: aws-cdk
description: Conventions for writing and validating AWS CDK infrastructure in TypeScript - app and stack layout, L2 constructs over raw CloudFormation, least-privilege IAM, stateful resource safety, and the synth/diff/test loop that must pass before any change is called done. Load before writing or changing anything under the CDK app, `cdk.json`, or a construct, and before running any `cdk` command.
---

CDK code is code that creates real infrastructure and real bills. Follow `write-code` for style; everything below is what CDK adds on top.

## 1. Before writing

- Read `cdk.json` for the app entry point, context, and feature flags. Never change a feature flag to make an error go away: flags change the generated template and can replace live resources.
- Read the existing stacks and constructs in full. Reuse a construct that already exists here before writing a new one, and copy this app's naming, props, and tagging conventions.
- Check the installed version (`npm ls aws-cdk-lib aws-cdk`) and ask the `docs-researcher` agent for the API at that version. Never write a construct's props from memory: they move between versions. That agent ships with the global config (`~/.claude/agents/`), not with this skill; if it is not available here, read the CDK API reference for that version yourself and say that you did.

## 2. App and stack layout

- `bin/` wires stacks together and nothing else: no resource definitions, no conditionals beyond environment selection.
- One stack per deployment lifetime. Resources that are created, updated, and destroyed together belong in the same stack; stateful storage that outlives the app belongs in its own.
- Reusable pieces are constructs in `lib/`, extending `Construct`, with an exported `readonly` props interface. Extend `Stack` only for stacks.
- Pass values through props. Never read `process.env` inside a construct, and never reach into another stack's internals; expose what callers need as a readonly property.
- Set `env` explicitly (account and region) on every stack. Environment-agnostic stacks silently disable `fromLookup` and AZ resolution.

## 3. Constructs

- Prefer L2 constructs (`Bucket`, `Function`) over L1 (`CfnBucket`). Drop to L1 or `addPropertyOverride` only when the L2 has no equivalent, and say in a comment which property forced it.
- Never hardcode ARNs, account IDs, bucket names, or region strings. Use the construct's own reference, `Stack.of(this).account` / `.region`, or a prop.
- Let CDK name physical resources. An explicit `bucketName`, `tableName`, or `functionName` blocks replacement updates and breaks a second deployment into the same account.
- Construct IDs are part of the logical ID. Renaming or re-parenting a construct destroys and recreates the resource; if a change does that to something stateful, stop and report it before going further.
- Tag at the app or stack level with `Tags.of(...)`, not per resource.

## 4. Security and safety

- Grant with the construct's own methods (`table.grantReadData(fn)`, `bucket.grantPut(fn)`). Write a raw `PolicyStatement` only when no grant exists, and never with `resources: ['*']` unless the action genuinely has no resource scope.
- No secrets in code or in environment variables at synth time. Use `Secret.fromSecretNameV2` or an SSM parameter and read it at runtime. `SecretValue.unsafePlainText` is never acceptable.
- Stateful resources (databases, buckets holding data, log groups you must keep) get `RemovalPolicy.RETAIN` outside of throwaway environments, and no `autoDeleteObjects` in production.
- Encryption on by default, public access blocked by default, and no `0.0.0.0/0` ingress except on a load balancer that is meant to be public.
- Lambda and container images pin a runtime the repo already uses; adding a new runtime version is a decision to raise, not to make silently.

## 5. Validate before calling it done

Run these in order. All must pass, and a failure is fixed here, not explained away.

1. `npx tsc --noEmit` (or the repo's build script) - type errors in CDK usually mean the API is being used wrong.
2. The repo's linter.
3. `npx cdk synth` - synthesis runs the code and catches missing context, circular references, and invalid props that types do not.
4. `npx cdk diff <stack>` - read it line by line. Every `[-]` on a stateful resource and every replacement (`replace` / `may be replaced`) must be intentional, and must be reported in your summary.
5. `npx cdk ls` if you added or renamed a stack, to confirm the app still enumerates.

Read the synthesized template for the changed resources: wildcard IAM actions or resources, public access, and missing encryption are easier to see there than in the CDK code.

## 6. Tests

Test with `aws-cdk-lib/assertions` against a synthesized template, in the repo's existing test layout.

- `Template.fromStack(stack)` plus `hasResourceProperties` for the properties that matter: the IAM policy, the encryption setting, the retention, the environment wiring.
- `resourceCountIs` for anything that must exist exactly once.
- Assert the security-relevant properties of every construct you add. A stack that synthesizes is not a stack that is correct.
- Snapshot tests catch unintended drift, but never update a snapshot without reading what changed in it.

## 7. Deployment

Never run `cdk deploy`, `cdk destroy`, `cdk import`, or `cdk bootstrap` on your own. Show the `cdk diff` and ask. If asked to deploy, name the target account, region, and stack back before running it.
