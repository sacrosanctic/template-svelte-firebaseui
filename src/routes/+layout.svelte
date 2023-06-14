<script>
	import './firebaseui.css'
	import {
		EmailAuthProvider,
		FacebookAuthProvider,
		getAuth,
		GithubAuthProvider,
		GoogleAuthProvider,
		PhoneAuthProvider,
		TwitterAuthProvider,
		onAuthStateChanged,
		signOut,
	} from 'firebase/auth'
	import { onMount, createEventDispatcher } from 'svelte'
	import { initializeApp, getApps } from 'firebase/app'
	import { browser } from '$app/environment'

	const firebaseConfig = {}

	const dispatch = createEventDispatcher()
	let ui
	let uiShown = false
	let container
	let uiConfig
	let firebaseApp = null
	let user
	let auth

	const getFirebaseApp = () => {
		if (firebaseApp) return firebaseApp

		if (getApps().length) {
			firebaseApp = getApps()[0]
			return firebaseApp
		}

		firebaseApp = initializeApp(firebaseConfig)

		auth = getAuth(firebaseApp)
		onAuthStateChanged(auth, (u) => (user = u))

		console.log(`${firebaseConfig.projectId} initialized on ${browser ? 'client' : 'server'}`)

		return firebaseApp
	}

	export let languageCode = 'en'
	export let signInWith = { google: true, emailPasswordless: true }
	export let tosUrl = undefined // '.../terms' | () => window.location.assign("your-terms-url");
	export let privacyPolicyUrl = undefined
	export let signInSuccessUrl = undefined
	export let continueUrl = undefined
	export let forceSameDevice = false

	onMount(async () => {
		// this is only for cdn
		// await import('https://www.gstatic.com/firebasejs/9.8.2/firebase-app-compat.js');
		// await import('https://www.gstatic.com/firebasejs/9.8.2/firebase-auth-compat.js');
		// await import(
		// 	`https://www.gstatic.com/firebasejs/ui/6.0.1/firebase-ui-auth__${languageCode}.js`
		// );

		// but this doesnt support other languages
		await import('firebaseui')

		loadConfig()
		initAuthUi()
	})

	const loadConfig = () => {
		uiConfig = {
			callbacks: {
				signInSuccessWithAuthResult: (authResult) => {
					dispatch('updateuserdata', {
						user: authResult.user,
						isNewUser: authResult.additionalUserInfo.isNewUser,
					}) // deprecated in favor of authresult
					dispatch('authresult', authResult)
					dispatch('success', 'auth success')

					console.log(authResult)
					return !!signInSuccessUrl // if  true uses first signInSuccessUrl parameter given in the URL then the default signInSuccessUrl given in config here; if false, page won't redirect automatically
				},
				signInFailure: (error) => {
					// Some unrecoverable error occurred during sign-in.
					// Return a promise when error handling is completed and FirebaseUI
					// will reset, clearing any UI. This commonly occurs for error code
					// 'firebaseui/anonymous-upgrade-merge-conflict' when merge conflict
					// occurs. Check below for more details on this.
					return handleUIError(error)
				},
				uiShown: () => (uiShown = true),
			},
			credentialHelper: firebaseui.auth.CredentialHelper.NONE, // disabling for moment if it makes harder with redirect (is ok if works through popup)
			signInFlow: 'redirect',
			signInOptions: [
				signInWith.google && GoogleAuthProvider.PROVIDER_ID,
				signInWith.facebook && FacebookAuthProvider.PROVIDER_ID,
				signInWith.twitter && TwitterAuthProvider.PROVIDER_ID,
				signInWith.github && GithubAuthProvider.PROVIDER_ID,
				signInWith.emailPassword && EmailAuthProvider.PROVIDER_ID,
				signInWith.emailPasswordless && {
					provider: EmailAuthProvider.PROVIDER_ID,
					signInMethod: EmailAuthProvider.EMAIL_LINK_SIGN_IN_METHOD,
					forceSameDevice,
					emailLinkSignIn: () => {
						return {
							url: continueUrl,
						}
					},
				},
				signInWith.phone && PhoneAuthProvider.PROVIDER_ID,
				signInWith.anonymous && firebaseui.auth.AnonymousAuthProvider.PROVIDER_ID,
				{
					provider: 'oidc.line',
					providerName: 'Line',
					// To override the full label of the button.
					// fullLabel: 'Employee Login',
					buttonColor: '#3bce15',
					iconUrl: 'https://cdn-icons-png.flaticon.com/512/124/124027.png',
					customParameters: { initial_amr_display: 'lineqr' },
				},
			],
			signInSuccessUrl,
			tosUrl,
			privacyPolicyUrl,
		}
	}

	const initAuthUi = () => {
		ui ? ui.reset() : (ui = new firebaseui.auth.AuthUI(getAuth(getFirebaseApp())))
		ui.start(container, uiConfig)
	}

	const handleUIError = async (error) => {
		console.error(error)
		window.location.replace('/')
	}
</script>

<svelte:head>
	{#if languageCode === 'iw' || languageCode === 'ar'}
		<link
			href="https://www.gstatic.com/firebasejs/ui/6.0.1/firebase-ui-auth-rtl.css"
			rel="stylesheet"
		/>
	{:else}
		<link
			href="https://www.gstatic.com/firebasejs/ui/6.0.1/firebase-ui-auth.css"
			rel="stylesheet"
		/>
	{/if}
</svelte:head>

<div bind:this={container} />

{#if !uiShown}
	<div>Loading...</div>
{/if}

{#if user}
	<button
		on:click={() => {
			signOut(auth)
			initAuthUi()
		}}>signout</button
	>
	<slot />
{/if}

<pre>{JSON.stringify(user, null, 2)}</pre>
