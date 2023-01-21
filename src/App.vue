<template>
	<div class="app">
		<div v-if="progress || task" class="content">
			<p>{{ task }}</p>
			<p>{{ progress }}% of current task</p>
			<template v-if="found">
				<p />
				<p class="result">{{ found }}</p>
				<button @click="reset">Click here to reset this application</button>
			</template>
		</div>
		<div v-else-if="!input" class="content">
			<h2>Arweave passphrase recovery service.</h2>
			<p />
			<p>Application to recover from an incomplete 11 words passphrase or a 12 words passphrase with a small ordering mistake or incorrect word.</p>
			<!-- <p>This tool will try to find the missing word. If you know where it is located, use "..." in its place.</p> -->
			<p>A fee of 5% of the wallet balance will automatically be paid on sucessful recovery. The amount charged will not exceed 100GB worth of arweave storage.</p>
			<template v-if="found">
				<p />
				<p class="result">{{ found }}</p>
				<button @click="reset">Click here to reset this application</button>
			</template>
		</div>
		<textarea v-model="input" :disabled="!!task" @keydown.enter.prevent="() => run()" autofocus></textarea>
	</div>
</template>



<script setup lang="ts">
// @ts-ignore
import { getKeyPairFromMnemonic } from 'human-crypto-keys'
import { validateMnemonic as validateM } from 'bip39-web-crypto'
import { ref, computed } from 'vue';
import Arweave from 'arweave'
import gql from 'arweave-graphql'

const res = {
	english: () => import('bip39-web-crypto/src/wordlists/english.json')
} as const

const gateway = 'https://arweave.net'
const gatewayGql = 'https://arweave.net/graphql'
const useWalletList = false
const arweave = new Arweave(urlToSettings(gateway))
const graphql = gql(gatewayGql)
const input = ref('')
const task = ref('')
const progress = ref('')
const setProgress = (n: number) => progress.value = '' + Math.floor(n * 100)
const foundArray = ref(JSON.parse(localStorage.getItem('found') ?? '[]') as any[])
const found = computed(() => foundArray.value.length ? foundArray.value : undefined)
const wordList = getWordList('english')

const payoutAddress = (async () => {
	return 'TId0Wix2KFl1gArtAT6Do1CbWU_0wneGvS5X9BfW5PE'
})()
const appAddress = (async () => {
	return location.pathname.split('/').find(e => e.length === 43) ?? location.origin
})()
const walletList = (async (): Promise<string[] | undefined> => {
	if (!useWalletList) { return }
	try { return (await arweave.api.get('wallet_list')).json().then(wallets => wallets.map((w: any) => w?.address)) } catch (e) {}
})()



async function run () {
	if (!input.value) { return alert('Type your passphrase') }
	const passphraseArray = input.value.split(' ')
	const words = await wordList
	const mistyped = passphraseArray.filter(e => e !== '...' && !words.includes(e))
	if (mistyped.length) { return alert(`The following words are mistyped or not part of the dictionary: \n\n${mistyped.join(', ')} \n\nNo tests were attempted and nothing was charged. Retry with the correct spelling if recovery is still necessary.`) }
	if (passphraseArray.length === 12) {
		input.value = ''
		if (passphraseArray.includes('...')) {
			await guessMissing(passphraseArray.map(e => e === '...' ? undefined : e))
		} else {
			for (let i = passphraseArray.length-2; i >= 0; i--) {
				setProgress((passphraseArray.length-2-i) / (passphraseArray.length-2))
				const current = swap(passphraseArray, i)
				task.value = current.join(' ') + ' | testing ordering'
				await testPassphrase(current)
			}
			for (let i = passphraseArray.length-1; i >= 0; i--) {
				const current = [...passphraseArray] as (string | undefined)[]
				current[i] = undefined
				task.value = current.map(e => e ?? '...').join(' ')
				await guessMissing(current)
			}
		}
	} else if (passphraseArray.length === 11) {
		input.value = ''
		passphraseArray.push(undefined as any)
		let current: (string | undefined)[] | undefined
		for (let i = passphraseArray.length-1; i >= 0; i--) {
			current = current ? swap(current, i) : passphraseArray
			task.value = current.map(e => e ?? '...').join(' ') + ` | testing word at position ${i + 1}`
			await guessMissing(current)
		}
	} else { alert('The tool was not made for the format you entered. Maybe there is a new version available elsewhere. Make sure that you trust the origin or the code of any application you provide your passphrase to, even if incomplete') }
}




type QueryAllTx = (...args: Parameters<typeof graphql['getTransactions']>) => Promise<Awaited<ReturnType<typeof graphql['getTransactions']>>['transactions']['edges']>
const queryAllTx: QueryAllTx = async (...args) => {
	const results = [] as Awaited<ReturnType<QueryAllTx>>
	const run = async (after?: string) => {
		const response = await graphql.getTransactions({ ...args[0], after })
		const edges = response.transactions.edges
		results.push(...edges)
		if (response.transactions.pageInfo.hasNextPage) { await run(edges[edges.length - 1].cursor) }
	}
	await run()
	return results
}

function urlToSettings (url: string) {
	const obj = new URL(url)
	const protocol = obj.protocol.replace(':', '')
	const host = obj.hostname
	const port = obj.port ? parseInt(obj.port) : protocol === 'https' ? 443 : 80
	return { protocol, host, port }
}

async function pkcs8ToJwk (key: Uint8Array) {
	const imported = await window.crypto.subtle.importKey('pkcs8', key, { name: 'RSA-PSS', hash: 'SHA-256' }, true, ['sign'])
	const jwk = await window.crypto.subtle.exportKey('jwk', imported)
	delete jwk.key_ops
	delete jwk.alg
	return jwk
}

async function derive (passphrase: string) {
	let keyPair = await getKeyPairFromMnemonic(passphrase, { id: 'rsa', modulusLength: 4096 }, { privateKeyFormat: 'pkcs8-der' })
	const jwk = await pkcs8ToJwk(keyPair.privateKey)
	return jwk
}

function swap <T> (passphrase: T[], i: number): T[] {
	const a = passphrase[i]
	const b = passphrase[i+1]
	const res = [...passphrase]
	res[i] = b
	res[i+1] = a
	return res
}

async function guessMissing (passphraseArray: (string | undefined)[]) {
	if (passphraseArray.length != 12) { throw `can't guess missing word in a passprase of length ${passphraseArray.length}` }
	const words = await wordList
	const i = passphraseArray.findIndex(e => !e)
	for (const word of words) {
		const passphrase = [...passphraseArray] as string[]
		passphrase[i] = word

		setProgress(words.indexOf(word) / words.length)
		await testPassphrase(passphrase)
	}
}

function pushResult (passphrase: string, address: string) {
	foundArray.value.push({ passphrase, address })
	localStorage.setItem('found', JSON.stringify(foundArray.value))
}

function reset () { localStorage.clear(); foundArray.value = [] }

async function testPassphrase (passphraseArray: string[]) {
	const passphrase = passphraseArray.join(' ')
	const valid = await validateM(passphrase, await wordList)
	if (!valid) { return }
	const jwk = await derive(passphrase) as any
	const address = await arweave.wallets.jwkToAddress(jwk)
	const list = await walletList
	const result = list ? list.includes(address)
		: await gql(gatewayGql).getTransactions({ recipients: [address] }).then(res => !!res.transactions.edges.length)
	if (result) { submit(passphrase, address, jwk) }
}

async function submit (passphrase: string, address: string, jwk: any) {
	const routine = async (): Promise<void> => { try {
		const [existingFeeTxs, balance, cap] = await Promise.all([
			queryAllTx({ recipients: await payoutAddress, owners: address }),
			arweave.wallets.getBalance(address),
			await arweave.transactions.getPrice(100000000000),
		])
		const settled = existingFeeTxs.filter(tx => +tx.node.quantity.winston > 0 && tx.node.block).map(tx => +tx.node.quantity.winston).reduce((acc, a) => acc + a, 0)
		const unsettled = existingFeeTxs.filter(tx => +tx.node.quantity.winston > 0 && !tx.node.block).map(tx => +tx.node.quantity.winston).reduce((acc, a) => acc + a, 0)
		const fee = Math.min(Math.floor(+balance / 20), +cap)
		const quantity = fee - settled - unsettled
		if (fee - settled <= 0) { return }
		if (quantity <= 0) {
			await new Promise(res => setTimeout(res, 60000))
			return routine()
		}
		const tx = await arweave.createTransaction({
			quantity: quantity.toString(),
			target: await payoutAddress,
		})
		Object.entries({
			app: 'Recovery fee',
		} as const).forEach(([name, value]) => tx.addTag(name, value))
		const currentLocation = await appAddress
		if (currentLocation) { tx.addTag('from', currentLocation) }
		await arweave.transactions.sign(tx, jwk)
		await arweave.transactions.post(tx)
		await new Promise(res => setTimeout(res, 180000))
		return routine()
	} catch (e) {
		console.error(e)
		await new Promise(res => setTimeout(res, 180000))
		return routine()
	} }
	await routine()
	pushResult(passphrase, address)
}



type ResAwaited = { [key in keyof typeof res]: Awaited<ReturnType<typeof res[key]>> }
async function getWordList <T extends keyof typeof res | undefined = undefined> (lang?: T) {
	const keys = (lang ? [lang] : Object.keys(res)) as [NonNullable<T>]
	const imports = await Promise.all(keys.map(async lang => ([lang, await res[lang]().then(i => i.default)] as const))).then(res => Object.fromEntries(res))
	const result = lang === undefined ? imports : imports[lang as string]
	return result as T extends keyof typeof res ? ResAwaited[T] : ResAwaited
}
</script>



<style scoped>
.app {
	width: 100vw;
	height: 100vh;
	box-sizing: border-box;
	--padding: 20vh 10vw;
	--max-width: 800px;
}

.content {
	padding: var(--padding);
	max-width: var(--max-width);
	margin: auto;
	display: flex;
	flex-direction: column;
}

h2,
p {
	margin-top: 0;
	font-size: 1em;
	font-weight: inherit;
}

textarea {
	padding: var(--padding);
	max-width: var(--max-width);
	margin: auto;
	display: flex;
	position: absolute;
	inset: 0;
	background: transparent;
	outline: 0;
	border: 0;
	font-size: 1em;
	font-family: inherit;
	color: inherit;
	resize: none;
}

button, .result {
	z-index: 1;
}
</style>
