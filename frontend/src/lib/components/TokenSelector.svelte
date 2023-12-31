<script lang="ts">
    import { ethers } from 'ethers';
    import { type Token } from '$lib/Models';
    import GenericERC20 from "$lib/contracts/GenericERC20.json";
    import fsm from 'svelte-fsm'
	import TokenView from './TokenView.svelte';
	import PendingSpinner from './PendingSpinner.svelte';
	import { provider } from '$lib/Stores';

    export let disabled: boolean;
    export let token: Token | undefined;
    export let valid: boolean;
    export let error: string | undefined;

    let address: string;
    let tx: Promise<any>;

    export const state = fsm('entering', {
        entering: {
            _enter() {
                error = undefined;
                token = undefined;
            },
            change: 'loading'
        },
        loading: {
            _enter() {
                address = address.trim();
                tx = getTokenDetails()
                  .then(this.loaded)
                  .catch(this.error)
            },

            loaded(details: Token) {
                token = details
                return 'valid'; 
            },

            error(e) {
				error = e.message;
                return 'invalid';
            }
        },
        valid: {
            input: 'entering'
        },
        invalid: {
            input: 'entering'
        }
    });

    $: valid = $state === 'valid';

    const getTokenDetails = async () => {
        if (!ethers.isAddress(address)) {
            throw new Error('not an address');
        }

        const tokenContract = new ethers.Contract(
            address,
            GenericERC20.abi,
            $provider
        );

        let name: string = await tokenContract.name();
        let symbol: string = await tokenContract.symbol();
        let decimals: number = await tokenContract.decimals();       
        
        return { name, symbol, decimals, address }; 
    }

    const clear = () => {
        address = '';
        token = undefined;
        error = undefined;
        state.input();
    }
</script>

<div>
    <input type="text" class={$state} bind:value={address} on:change={state.change} on:input={state.input} {disabled} placeholder="Enter a token address..." />
    <PendingSpinner message="" {tx} />
    {#if token}
        <TokenView name={token.name} symbol={token.symbol} on:clear={clear} />
    {/if}
</div>

<style>
    div {
        display: flex;
        align-items: center;
        gap: 0.2em;
    }
    input {
        flex-grow: 1;
    }
</style>

