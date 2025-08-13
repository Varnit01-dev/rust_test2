use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    msg,
    program::{invoke, invoke_signed},
    program_error::ProgramError,
    program_pack::Pack,
    pubkey::Pubkey,
    rent::Rent,
    system_instruction,
    sysvar::{clock::Clock, Sysvar},
};
use spl_token::{
    instruction as token_instruction,
    state::{Account as TokenAccount, Mint},
};
use borsh::{BorshDeserialize, BorshSerialize};

// Instructions for the swap program
#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub enum SwapInstruction {
    /// Initialize a new swap pool
    /// Accounts expected:
    /// 0. `[signer]` Pool creator
    /// 1. `[writable]` Swap pool account
    /// 2. `[]` Pool authority (PDA)
    /// 3. `[]` Token A mint
    /// 4. `[]` Token B mint
    /// 5. `[writable]` Pool token A account
    /// 6. `[writable]` Pool token B account
    /// 7. `[writable]` Pool LP token mint
    /// 8. `[writable]` Pool LP fee account
    /// 9. `[]` Token program
    /// 10. `[]` System program
    /// 11. `[]` Rent sysvar
    Initialize {
        nonce: u8,
        fee_numerator: u64,
        fee_denominator: u64,
    },

    /// Swap tokens
    /// Accounts expected:
    /// 0. `[]` Swap pool account
    /// 1. `[]` Pool authority
    /// 2. `[signer]` User authority
    /// 3. `[writable]` User source token account
    /// 4. `[writable]` Pool source token account
    /// 5. `[writable]` Pool destination token account
    /// 6. `[writable]` User destination token account
    /// 7. `[writable]` Pool LP fee account
    /// 8. `[]` Token program
    Swap {
        amount_in: u64,
        minimum_amount_out: u64,
    },

    /// Add liquidity to pool
    /// Accounts expected:
    /// 0. `[]` Swap pool account
    /// 1. `[]` Pool authority
    /// 2. `[signer]` User authority
    /// 3. `[writable]` User token A account
    /// 4. `[writable]` User token B account
    /// 5. `[writable]` Pool token A account
    /// 6. `[writable]` Pool token B account
    /// 7. `[writable]` Pool LP token mint
    /// 8. `[writable]` User LP token account
    /// 9. `[]` Token program
    AddLiquidity {
        pool_token_amount: u64,
        maximum_token_a_amount: u64,
        maximum_token_b_amount: u64,
    },

    /// Remove liquidity from pool
    /// Accounts expected:
    /// 0. `[]` Swap pool account
    /// 1. `[]` Pool authority
    /// 2. `[signer]` User authority
    /// 3. `[writable]` Pool LP token mint
    /// 4. `[writable]` User LP token account
    /// 5. `[writable]` Pool token A account
    /// 6. `[writable]` Pool token B account
    /// 7. `[writable]` User token A account
    /// 8. `[writable]` User token B account
    /// 9. `[writable]` Pool LP fee account
    /// 10. `[]` Token program
    RemoveLiquidity {
        pool_token_amount: u64,
        minimum_token_a_amount: u64,
        minimum_token_b_amount: u64,
    },
}

// Swap pool state
#[derive(BorshSerialize, BorshDeserialize, Debug, Clone)]
pub struct SwapPool {
    /// Version of the swap pool
    pub version: u8,
    /// Bump seed used to generate the authority
    pub nonce: u8,
    /// Token A mint
    pub token_a: Pubkey,
    /// Token B mint  
    pub token_b: Pubkey,
    /// Pool tokens are issued when A or B tokens are deposited
    pub pool_mint: Pubkey,
    /// Token A account for the pool
    pub token_a_account: Pubkey,
    /// Token B account for the pool
    pub token_b_account: Pubkey,
    /// Pool token account to receive fees
    pub pool_fee_account: Pubkey,
    /// Fee numerator
    pub fee_numerator: u64,
    /// Fee denominator  
    pub fee_denominator: u64,
    /// Owner trading fee numerator
    pub owner_trade_fee_numerator: u64,
    /// Owner trading fee denominator
    pub owner_trade_fee_denominator: u64,
    /// Owner withdraw fee numerator
    pub owner_withdraw_fee_numerator: u64,
    /// Owner withdraw fee denominator
    pub owner_withdraw_fee_denominator: u64,
}

impl SwapPool {
    pub const LEN: usize = 324; // Calculated size for the struct
}

entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    instruction_data: &[u8],
) -> ProgramResult {
    let instruction = SwapInstruction::try_from_slice(instruction_data)
        .map_err(|_| ProgramError::InvalidInstructionData)?;

    match instruction {
        SwapInstruction::Initialize { nonce, fee_numerator, fee_denominator } => {
            process_initialize(program_id, accounts, nonce, fee_numerator, fee_denominator)
        }
        SwapInstruction::Swap { amount_in, minimum_amount_out } => {
            process_swap(program_id, accounts, amount_in, minimum_amount_out)
        }
        SwapInstruction::AddLiquidity { 
            pool_token_amount, 
            maximum_token_a_amount, 
            maximum_token_b_amount 
        } => {
            process_add_liquidity(
                program_id, 
                accounts, 
                pool_token_amount, 
                maximum_token_a_amount, 
                maximum_token_b_amount
            )
        }
        SwapInstruction::RemoveLiquidity { 
            pool_token_amount, 
            minimum_token_a_amount, 
            minimum_token_b_amount 
        } => {
            process_remove_liquidity(
                program_id, 
                accounts, 
                pool_token_amount, 
                minimum_token_a_amount, 
                minimum_token_b_amount
            )
        }
    }
}

pub fn process_initialize(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    nonce: u8,
    fee_numerator: u64,
    fee_denominator: u64,
) -> ProgramResult {
    let account_info_iter = &mut accounts.iter();
    let pool_creator_info = next_account_info(account_info_iter)?;
    let swap_pool_info = next_account_info(account_info_iter)?;
    let authority_info = next_account_info(account_info_iter)?;
    let token_a_mint_info = next_account_info(account_info_iter)?;
    let token_b_mint_info = next_account_info(account_info_iter)?;
    let token_a_account_info = next_account_info(account_info_iter)?;
    let token_b_account_info = next_account_info(account_info_iter)?;
    let pool_mint_info = next_account_info(account_info_iter)?;
    let pool_fee_account_info = next_account_info(account_info_iter)?;
    let token_program_info = next_account_info(account_info_iter)?;
    let system_program_info = next_account_info(account_info_iter)?;
    let rent_info = next_account_info(account_info_iter)?;

    if !pool_creator_info.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }

    // Verify the authority is a valid PDA
    let authority_signer_seeds = &[
        &swap_pool_info.key.to_bytes()[..32],
        &[nonce],
    ];
    let (expected_authority, _) = Pubkey::find_program_address(authority_signer_seeds, program_id);
    if expected_authority != *authority_info.key {
        return Err(ProgramError::InvalidArgument);
    }

    // Create swap pool account
    let rent = Rent::from_account_info(rent_info)?;
    let pool_account_rent = rent.minimum_balance(SwapPool::LEN);

    invoke(
        &system_instruction::create_account(
            pool_creator_info.key,
            swap_pool_info.key,
            pool_account_rent,
            SwapPool::LEN as u64,
            program_id,
        ),
        &[
            pool_creator_info.clone(),
            swap_pool_info.clone(),
            system_program_info.clone(),
        ],
    )?;

    // Initialize swap pool state
    let swap_pool = SwapPool {
        version: 1,
        nonce,
        token_a: *token_a_mint_info.key,
        token_b: *token_b_mint_info.key,
        pool_mint: *pool_mint_info.key,
        token_a_account: *token_a_account_info.key,
        token_b_account: *token_b_account_info.key,
        pool_fee_account: *pool_fee_account_info.key,
        fee_numerator,
        fee_denominator,
        owner_trade_fee_numerator: 0,
        owner_trade_fee_denominator: 0,
        owner_withdraw_fee_numerator: 0,
        owner_withdraw_fee_denominator: 0,
    };

    swap_pool.serialize(&mut &mut swap_pool_info.data.borrow_mut()[..])?;

    msg!("Swap pool initialized");
    Ok(())
}

pub fn process_swap(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    amount_in: u64,
    minimum_amount_out: u64,
) -> ProgramResult {
    let account_info_iter = &mut accounts.iter();
    let swap_pool_info = next_account_info(account_info_iter)?;
    let authority_info = next_account_info(account_info_iter)?;
    let user_authority_info = next_account_info(account_info_iter)?;
    let user_source_info = next_account_info(account_info_iter)?;
    let pool_source_info = next_account_info(account_info_iter)?;
    let pool_destination_info = next_account_info(account_info_iter)?;
    let user_destination_info = next_account_info(account_info_iter)?;
    let pool_fee_account_info = next_account_info(account_info_iter)?;
    let token_program_info = next_account_info(account_info_iter)?;

    if !user_authority_info.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }

    let swap_pool = SwapPool::try_from_slice(&swap_pool_info.data.borrow())?;

    // Get current pool balances
    let pool_source_account = TokenAccount::unpack(&pool_source_info.data.borrow())?;
    let pool_dest_account = TokenAccount::unpack(&pool_destination_info.data.borrow())?;

    // Calculate swap amount using constant product formula (x * y = k)
    let source_amount = pool_source_account.amount;
    let dest_amount = pool_dest_account.amount;

    // Apply fee
    let fee_amount = amount_in
        .checked_mul(swap_pool.fee_numerator)
        .ok_or(ProgramError::ArithmeticOverflow)?
        .checked_div(swap_pool.fee_denominator)
        .ok_or(ProgramError::ArithmeticOverflow)?;

    let swap_amount = amount_in
        .checked_sub(fee_amount)
        .ok_or(ProgramError::ArithmeticOverflow)?;

    // Calculate output using AMM formula: dy = (y * dx) / (x + dx)
    let invariant = source_amount
        .checked_mul(dest_amount)
        .ok_or(ProgramError::ArithmeticOverflow)?;

    let new_source_amount = source_amount
        .checked_add(swap_amount)
        .ok_or(ProgramError::ArithmeticOverflow)?;

    let new_dest_amount = invariant
        .checked_div(new_source_amount)
        .ok_or(ProgramError::ArithmeticOverflow)?;

    let amount_out = dest_amount
        .checked_sub(new_dest_amount)
        .ok_or(ProgramError::ArithmeticOverflow)?;

    if amount_out < minimum_amount_out {
        msg!("Slippage exceeded");
        return Err(ProgramError::InvalidArgument);
    }

    // Transfer source tokens from user to pool
    invoke(
        &token_instruction::transfer(
            token_program_info.key,
            user_source_info.key,
            pool_source_info.key,
            user_authority_info.key,
            &[],
            amount_in,
        )?,
        &[
            user_source_info.clone(),
            pool_source_info.clone(),
            user_authority_info.clone(),
            token_program_info.clone(),
        ],
    )?;

    // Transfer destination tokens from pool to user
    let authority_signer_seeds = &[
        &swap_pool_info.key.to_bytes()[..32],
        &[swap_pool.nonce],
    ];

    invoke_signed(
        &token_instruction::transfer(
            token_program_info.key,
            pool_destination_info.key,
            user_destination_info.key,
            authority_info.key,
            &[],
            amount_out,
        )?,
        &[
            pool_destination_info.clone(),
            user_destination_info.clone(),
            authority_info.clone(),
            token_program_info.clone(),
        ],
        &[authority_signer_seeds],
    )?;

    msg!("Swapped {} for {}", amount_in, amount_out);
    Ok(())
}

pub fn process_add_liquidity(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    pool_token_amount: u64,
    maximum_token_a_amount: u64,
    maximum_token_b_amount: u64,
) -> ProgramResult {
    let account_info_iter = &mut accounts.iter();
    let swap_pool_info = next_account_info(account_info_iter)?;
    let authority_info = next_account_info(account_info_iter)?;
    let user_authority_info = next_account_info(account_info_iter)?;
    let user_token_a_info = next_account_info(account_info_iter)?;
    let user_token_b_info = next_account_info(account_info_iter)?;
    let pool_token_a_info = next_account_info(account_info_iter)?;
    let pool_token_b_info = next_account_info(account_info_iter)?;
    let pool_mint_info = next_account_info(account_info_iter)?;
    let user_pool_token_info = next_account_info(account_info_iter)?;
    let token_program_info = next_account_info(account_info_iter)?;

    if !user_authority_info.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }

    let swap_pool = SwapPool::try_from_slice(&swap_pool_info.data.borrow())?;

    // Get current pool token balances
    let pool_token_a_account = TokenAccount::unpack(&pool_token_a_info.data.borrow())?;
    let pool_token_b_account = TokenAccount::unpack(&pool_token_b_info.data.borrow())?;
    let pool_mint = Mint::unpack(&pool_mint_info.data.borrow())?;

    // Calculate required token amounts to maintain ratio
    let pool_token_a_amount = pool_token_a_account.amount;
    let pool_token_b_amount = pool_token_b_account.amount;

    let (token_a_amount, token_b_amount) = if pool_mint.supply == 0 {
        // Initial liquidity
        (maximum_token_a_amount, maximum_token_b_amount)
    } else {
        // Calculate proportional amounts
        let token_a_amount = pool_token_amount
            .checked_mul(pool_token_a_amount)
            .ok_or(ProgramError::ArithmeticOverflow)?
            .checked_div(pool_mint.supply)
            .ok_or(ProgramError::ArithmeticOverflow)?;

        let token_b_amount = pool_token_amount
            .checked_mul(pool_token_b_amount)
            .ok_or(ProgramError::ArithmeticOverflow)?
            .checked_div(pool_mint.supply)
            .ok_or(ProgramError::ArithmeticOverflow)?;

        if token_a_amount > maximum_token_a_amount || token_b_amount > maximum_token_b_amount {
            msg!("Liquidity amount exceeds maximum");
            return Err(ProgramError::InvalidArgument);
        }

        (token_a_amount, token_b_amount)
    };

    let authority_signer_seeds = &[
        &swap_pool_info.key.to_bytes()[..32],
        &[swap_pool.nonce],
    ];

    // Transfer token A from user to pool
    invoke(
        &token_instruction::transfer(
            token_program_info.key,
            user_token_a_info.key,
            pool_token_a_info.key,
            user_authority_info.key,
            &[],
            token_a_amount,
        )?,
        &[
            user_token_a_info.clone(),
            pool_token_a_info.clone(),
            user_authority_info.clone(),
            token_program_info.clone(),
        ],
    )?;

    // Transfer token B from user to pool
    invoke(
        &token_instruction::transfer(
            token_program_info.key,
            user_token_b_info.key,
            pool_token_b_info.key,
            user_authority_info.key,
            &[],
            token_b_amount,
        )?,
        &[
            user_token_b_info.clone(),
            pool_token_b_info.clone(),
            user_authority_info.clone(),
            token_program_info.clone(),
        ],
    )?;

    // Mint pool tokens to user
    invoke_signed(
        &token_instruction::mint_to(
            token_program_info.key,
            pool_mint_info.key,
            user_pool_token_info.key,
            authority_info.key,
            &[],
            pool_token_amount,
        )?,
        &[
            pool_mint_info.clone(),
            user_pool_token_info.clone(),
            authority_info.clone(),
            token_program_info.clone(),
        ],
        &[authority_signer_seeds],
    )?;

    msg!("Added liquidity: {} LP tokens minted", pool_token_amount);
    Ok(())
}

pub fn process_remove_liquidity(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    pool_token_amount: u64,
    minimum_token_a_amount: u64,
    minimum_token_b_amount: u64,
) -> ProgramResult {
    let account_info_iter = &mut accounts.iter();
    let swap_pool_info = next_account_info(account_info_iter)?;
    let authority_info = next_account_info(account_info_iter)?;
    let user_authority_info = next_account_info(account_info_iter)?;
    let pool_mint_info = next_account_info(account_info_iter)?;
    let user_pool_token_info = next_account_info(account_info_iter)?;
    let pool_token_a_info = next_account_info(account_info_iter)?;
    let pool_token_b_info = next_account_info(account_info_iter)?;
    let user_token_a_info = next_account_info(account_info_iter)?;
    let user_token_b_info = next_account_info(account_info_iter)?;
    let pool_fee_account_info = next_account_info(account_info_iter)?;
    let token_program_info = next_account_info(account_info_iter)?;

    if !user_authority_info.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }

    let swap_pool = SwapPool::try_from_slice(&swap_pool_info.data.borrow())?;

    // Get current balances
    let pool_token_a_account = TokenAccount::unpack(&pool_token_a_info.data.borrow())?;
    let pool_token_b_account = TokenAccount::unpack(&pool_token_b_info.data.borrow())?;
    let pool_mint = Mint::unpack(&pool_mint_info.data.borrow())?;

    // Calculate withdrawal amounts
    let token_a_amount = pool_token_amount
        .checked_mul(pool_token_a_account.amount)
        .ok_or(ProgramError::ArithmeticOverflow)?
        .checked_div(pool_mint.supply)
        .ok_or(ProgramError::ArithmeticOverflow)?;

    let token_b_amount = pool_token_amount
        .checked_mul(pool_token_b_account.amount)
        .ok_or(ProgramError::ArithmeticOverflow)?
        .checked_div(pool_mint.supply)
        .ok_or(ProgramError::ArithmeticOverflow)?;

    if token_a_amount < minimum_token_a_amount || token_b_amount < minimum_token_b_amount {
        msg!("Withdrawal amount below minimum");
        return Err(ProgramError::InvalidArgument);
    }

    let authority_signer_seeds = &[
        &swap_pool_info.key.to_bytes()[..32],
        &[swap_pool.nonce],
    ];

    // Burn pool tokens
    invoke(
        &token_instruction::burn(
            token_program_info.key,
            user_pool_token_info.key,
            pool_mint_info.key,
            user_authority_info.key,
            &[],
            pool_token_amount,
        )?,
        &[
            user_pool_token_info.clone(),
            pool_mint_info.clone(),
            user_authority_info.clone(),
            token_program_info.clone(),
        ],
    )?;

    // Transfer token A from pool to user
    invoke_signed(
        &token_instruction::transfer(
            token_program_info.key,
            pool_token_a_info.key,
            user_token_a_info.key,
            authority_info.key,
            &[],
            token_a_amount,
        )?,
        &[
            pool_token_a_info.clone(),
            user_token_a_info.clone(),
            authority_info.clone(),
            token_program_info.clone(),
        ],
        &[authority_signer_seeds],
    )?;

    // Transfer token B from pool to user
    invoke_signed(
        &token_instruction::transfer(
            token_program_info.key,
            pool_token_b_info.key,
            user_token_b_info.key,
            authority_info.key,
            &[],
            token_b_amount,
        )?,
        &[
            pool_token_b_info.clone(),
            user_token_b_info.clone(),
            authority_info.clone(),
            token_program_info.clone(),
        ],
        &[authority_signer_seeds],
    )?;

    msg!("Removed liquidity: {} tokens A, {} tokens B", token_a_amount, token_b_amount);
    Ok(())
}