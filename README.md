# Pre-Req Vault — Solution

Fork of the [prerequisite vault challenge](https://github.com/ShrinathNR/pre-req-vault), with the `withdraw` instruction extended to register the caller via CPI to the course's registration program.

## What's in this repo

- **Task 1** — understanding the vault program (instructions, accounts, state, flow) is written up in [`ARCHITECTURE.md`](./ARCHITECTURE.md).
- **Task 2** — `withdraw` now performs a CPI to the registration program's `initialize` instruction, recording a GitHub username. See below for the exact change and the on-chain proof.
- **Task 3** — the architecture diagram is in [`ARCHITECTURE.md`](./ARCHITECTURE.md): a component/account map and a sequence diagram, in standard notation.
- **Task 4** — video walkthrough script and deeper first-principles notes are in [`NOTES_FOR_VIDEO.md`](./NOTES_FOR_VIDEO.md).

## Task 2 — what was added

`withdraw` already declared (but never used) an `application_account` and
`application_program`. The extension, in
[`programs/pre-req-vault/src/instructions/withdraw.rs`](./programs/pre-req-vault/src/instructions/withdraw.rs):

```rust
// CPI to the application program to initialize your application account for registration.
let cpi_accounts = Initialize {
    user: self.user.to_account_info(),
    account: self.application_account.to_account_info(),
    system_program: self.system_program.to_account_info(),
};

let cpi_ctx = CpiContext::new(self.application_program.key(), cpi_accounts);

initialize(cpi_ctx, "kaushalya4s5s7".to_string())?;
```

No signer seeds are needed here — the user already signed the outer
`withdraw` transaction, and that signature carries through the CPI, so the
registration program sees the user as a real signer one level down. This is
different from the SOL transfer earlier in the same instruction, where the
`Vault` PDA has no private key and the program has to re-supply its seeds
to prove authority instead.

### Deployment

Deployed under a **program ID controlled by this wallet**, not the
staff-provided one — required so the CPI can't be mistaken for a call
through the course's own program:

- Program ID: `EmSSSZn3u5MTASmtJpb4XKiYjBNqLgDmdAEyUddUWpRs`
- Cluster: devnet
- Upgrade authority: this wallet (`SVjv1wQqMjkxUR3ABoufN4AKQpr2k2Kyc41x5FYTWHU`)

### Verification

`anchor test` passes end to end (initialize → deposit → withdraw → close).
On the `withdraw` transaction specifically:

- SOL transfer: `Vault` PDA balance goes from 0.5 SOL lower, confirming the
  withdrawal.
- Registration CPI: the registration PDA (`prereqs` + wallet, owned by the
  registration program) goes from **not existing** to holding the GitHub id
  `kaushalya4s5s7`, in the same transaction.

Transaction signatures (devnet):

| Step | Signature |
|---|---|
| initialize | `4cTQagLLzpLZXqrPZifDSVGXSCwdk5wWiVxWqz2A7HPwqK1VweRP4rgPANmZJYCpWjTnmHgHcVD2PJgcUMYzZQZX` |
| deposit | `KjS2MyfM9KbCSzzakV7d2hj1drmtYq5uf246YaRN5doGuCKgkPaF1fNCuDihdq4f4yWMh8piBMCuwDmhHmtBu7b` |
| withdraw (CPI here) | `2XnMrU7wohcHunBSxBc8V2rzjvbPSNZg6J95GsUPTdgH1Av1TBT6GGzhFXHGjoHFFZXXkdnuXKFptrbb1v7DDA5y` |

> Note: the registration program allows one registration per wallet, so a
> fresh `anchor test` run will fail at the `withdraw` step on this wallet —
> that's the program correctly rejecting a duplicate registration, not a
> bug. The signatures above are the original successful run.

## Running it yourself

```bash
pnpm install
anchor build
anchor deploy --provider.cluster devnet   # requires your own program ID via `anchor keys sync`
anchor test --skip-build --skip-deploy
```

Requires Rust, the Solana CLI, and Anchor 1.1.2 (`avm install 1.1.2 && avm use 1.1.2`) to match this workspace's `anchor-lang` version.
