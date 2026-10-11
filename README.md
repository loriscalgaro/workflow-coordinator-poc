# GitHub Free canonical-authority proof of concept

**PUBLIC DUMMY DATA ONLY. NOT PRODUCTION-APPROVED.**

This PoC makes a protected Git `main` branch the canonical store. It tests whether GitHub rejects direct main pushes, outdated state, duplicate operation identifiers, and changes not validated by trusted-base workflow code. There is no interaction with your C26/P11 workflow or ChatGPT Library.

## Critical limitations

- Use a **PUBLIC personal GitHub repository** for branch protection under GitHub Free. Do not copy private workflow data, tokens, real chemical models, or personal details into this repository.
- This is **not** a separately authenticated, fully automated sole coordinator. The repository owner is the human merger; protected main + strict checks provide server-side acceptance gates. The owner/admin can intentionally reconfigure protections; we do not claim to defend against a malicious administrator.
- `pull_request_target` runs a trusted-base Python validator and reads the proposed JSON as untrusted data through the GitHub API. It never checks out or executes proposed code. The workflow grants `statuses: write` only so the trusted check can post `canonical-gate/verified` on the PR head commit.
- Only same-repository PRs touching exactly `state/workflows/demo.json` are supported in this PoC. Fork PRs are deliberately not supported.
- `Canonical history audit` runs after a commit; it is detection, not the merge gate.
- The protected branch MUST require `canonical-gate/verified` with strict up-to-date checking. Without protection, the workflow does not enforce merges.

## 1. Create the repository

1. In a browser sign in to https://github.com.
2. Open https://github.com/new.
3. Repository name: **workflow-coordinator-poc**.
4. Choose **Public**. Keep "Add a README", `.gitignore`, and license **unchecked** (empty repository).
5. Select **Create repository**. Copy the displayed repository URL (`https://github.com/loriscalgaro/workflow-coordinator-poc.git`).
6. On your computer, extract this package so it contains `.github/workflows`, `scripts`, `state`, `tests`, and `README.md`. Ensure hidden `.github` directory is extracted.
7. Install Git (Git Bash on Windows) and Python 3 if you do not have them. In Git Bash type `git --version` and `python --version`.
8. Optionally run the 18 local self-tests with `python -m unittest discover -s tests -v`. In Git Bash change into the extracted folder and run:

```bash
 git init -b main
 git config user.name "YOUR_GITHUB_USERNAME"
 git config user.email "YOUR_GITHUB_NOREPLY_EMAIL"
 git add .
 git commit -m "Initialize canonical authority POC"
 git remote add origin https://github.com/YOUR_USERNAME/workflow-coordinator-poc.git
 git push -u origin main
```

GitHub may ask you to sign in through a credential manager. Do not paste passwords or access tokens into chat. The username/email values are local commit metadata; any GitHub-verified email or GitHub no-reply address is suitable.

## 2. Set Actions permissions

In repository **Settings > Actions > General**, under **Workflow permissions** select **Read repository contents permission** (or the most restrictive read-only default). Leave **Allow GitHub Actions to create and approve pull requests** disabled. Save.

The trusted gate workflow requests only `contents: read`, `pull-requests: read`, and `statuses: write`; it has no canonical content write permission.

## 3. Open an initial valid PR to register the required check

In Git Bash in the repository folder:

```bash
 git fetch origin
 git switch -c proposal/op-a-001 origin/main
 python scripts/create_proposal.py --operation op-a-001 --note "First public dummy transition"
 git add state/workflows/demo.json
 git commit -m "Propose op-a-001"
 git push -u origin proposal/op-a-001
```

In the GitHub repository: **Pull requests > New pull request**, base `main`, compare `proposal/op-a-001`, then **Create pull request**. Wait for the **Trusted canonical gate** workflow. The validator posts a commit status named **`canonical-gate/verified`** to the proposal head. This should be successful. **DO NOT MERGE YET.**

You can open **Actions** to inspect `Trusted canonical gate > publish-trusted-status` and confirm a successful run with an authenticated GitHub status. If no check appears, do not proceed until the workflow logs and token permissions are fixed.

## 4. Protect the canonical main branch BEFORE merging

Go to repository **Settings > Rules > Rulesets > New ruleset > New branch ruleset** (the layout may say **Settings > Rulesets**). Name it `canonical-main-enforcement`. Set **Enforcement status = Active**. Under **Target branches**, choose **Include default branch** (main).

Enable:

- **Require a pull request before merging**. Set required approving reviews to **0** for a one-person PoC.
- **Require status checks to pass** and add **`canonical-gate/verified`** (exact spelling). Select **GitHub Actions** as the expected status source. If you can only select "Any source", record this as an incomplete security gate and do **not** claim a full PoC PASS. **Require branches to be up to date before merging** (strict checking).
- **Require linear history**.
- **Block force pushes and branch deletion** (leave any force-push/delete allowances disabled).
- **Do not allow bypassing** if presented; use no bypass exemptions. Administrators who can edit rules still remain trusted in this PoC.

Save the **Active** ruleset. If settings are not available in your plan, STOP; do not substitute a private repository without equivalent enforcement.

Go to **Settings > General > Pull Requests** and enable **Allow squash merging**; disable **Allow merge commits** and **Allow rebase merging** for this PoC. This allows the post-merge auditor to verify one canonical state transition per accepted commit.

Verify the ruleset is Active and targets `main` before proceeding.

## 5. Test a competing proposal from the same genesis

Before merging `op-a-001`, from Git Bash run:

```bash
 git fetch origin
 git switch -c proposal/op-b-002 origin/main
 python scripts/create_proposal.py --operation op-b-002 --note "Competing transition from revision zero"
 git add state/workflows/demo.json
 git commit -m "Propose op-b-002"
 git push -u origin proposal/op-b-002
```

Open a second PR with base `main` and compare `proposal/op-b-002`. Both PRs can be green while `main` is still at revision zero. This is expected: merely passing validation is not a committed ownership claim.

Now merge **only op-a-001** using **Squash and merge**. The protected `main` branch should advance to revision 1. Run/inspect **Canonical history audit** for its PASS result.

Return to the **op-b-002** PR. Its branch is now out of date and must be **blocked by strict up-to-date checks**. Its `previous_commit` also refers to the old main commit. Do not force the second merge. A correct update would need a fresh proposal with a new parent and revised state.

**Important:** GitHub cannot guarantee the order in which two Actions jobs start. The atomic accepted main update and strict merge requirements, not Actions execution order, are the decisive part of this test.

## 6. Test duplicate replay rejection

```bash
 git fetch origin
 git switch -c proposal/replay-test origin/main
 python scripts/create_proposal.py --operation op-a-001 --note "Try replay"
```

Expected: the local helper rejects `op-a-001` because it already exists in the canonical journal. To test the GitHub server independently, while you are still on `proposal/replay-test`, run the *test-only* deliberately invalid proposal helper:

```bash
 python tests/make_invalid_proposal.py --case replay --operation op-a-001
 git add state/workflows/demo.json
 git commit -m "Negative test: replay"
 git push -u origin proposal/replay-test
```

Open a PR from `proposal/replay-test` to `main`. The trusted status must be **failure**, and GitHub must block merging. Do not merge negative-test PRs.

For a second negative test (stale predecessor), start a fresh branch from `origin/main`, run `python tests/make_invalid_proposal.py --case stale --operation op-c-003`, commit/push and open another PR. That must also fail.

## 7. Test bypass and direct-push rejection

With a proposal branch checked out and a commit already made on that branch, run:

```bash
 git push origin HEAD:main
```

Expected: the GitHub server rejects the direct push because `main` requires PR and status checks. **If this succeeds, the proof of concept FAILS immediately**. Do not trust the repository protections.

To test changed-file validation, add any second file to a new proposal PR. The trusted gate should post failure because only the JSON state file may change.

## 8. Evidence to retain

Save or copy the URLs and screenshots for: (a) the active ruleset configuration; (b) both PRs and the rejected stale PR; (c) the replay-rejected PR; (d) the rejected direct push; (e) the successful canonical history audit and the final main commit SHA. **Don't share authentication tokens.**

## GO / NO-GO

A `PASS` for this synthetic PoC requires the real GitHub server to enforce all protections above, including a required status from the expected GitHub Actions source, strict up-to-date checks, nonbypassable PR rules, and one accepted canonical update. The supplied local tests alone do not prove enforcement. A human repository owner can reconfigure rules, so this does not establish adversarial administrator resistance or a fully unattended workflow coordinator. It does not automatically approve the prior V04.2 branch, ChatGPT Files native routing, or any production workflow migration.
