## Complete Discord Quest (ENG)
 
> [!NOTE]
> This no longer works in browser!
> 


> [!WARNING]
> There are now two quest types ("stream" and "play")! Pay attention to the instructions!
>

How to use this script:
1. Accept a quest under User Settings -> Gift Inventory
2. Press <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>I</kbd> to open DevTools
3. Go to the `Console` tab
4. Paste the following code and hit enter:
<details>
	<summary>Click to expand</summary>
	
```js
delete window.$;
let wpRequire = webpackChunkdiscord_app.push([[Symbol()], {}, r => r]);
webpackChunkdiscord_app.pop();

let ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getStreamerActiveStreamMetadata).exports.Z;
let RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.ZP?.getRunningGames).exports.ZP;
let QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getQuest).exports.Z;
let ChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getAllThreadsForParent).exports.Z;
let GuildChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.ZP?.getSFWDefaultChannel).exports.ZP;
let FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.flushWaitQueue).exports.Z;
let api = Object.values(wpRequire.c).find(x => x?.exports?.tn?.get).exports.tn;

let quest = [...QuestsStore.quests.values()].find(x => x.id !== "1248385850622869556" && x.userStatus?.enrolledAt && !x.userStatus?.completedAt && new Date(x.config.expiresAt).getTime() > Date.now())
let isApp = typeof DiscordNative !== "undefined"
if(!quest) {
	console.log("You don't have any uncompleted quests!")
} else {
	const pid = Math.floor(Math.random() * 30000) + 1000
	
	const applicationId = quest.config.application.id
	const applicationName = quest.config.application.name
	const taskName = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"].find(x => quest.config.taskConfig.tasks[x] != null)
	const secondsNeeded = quest.config.taskConfig.tasks[taskName].target
	let secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0

	if(taskName === "WATCH_VIDEO" || taskName === "WATCH_VIDEO_ON_MOBILE") {
		const maxFuture = 10, speed = 7, interval = 1
		const enrolledAt = new Date(quest.userStatus.enrolledAt).getTime()
		let fn = async () => {			
			while(true) {
				const maxAllowed = Math.floor((Date.now() - enrolledAt)/1000) + maxFuture
				const diff = maxAllowed - secondsDone
				const timestamp = secondsDone + speed
				if(diff >= speed) {
					await api.post({url: `/quests/${quest.id}/video-progress`, body: {timestamp: Math.min(secondsNeeded, timestamp + Math.random())}})
					secondsDone = Math.min(secondsNeeded, timestamp)
				}
				
				if(timestamp >= secondsNeeded) {
					break
				}
				await new Promise(resolve => setTimeout(resolve, interval * 1000))
			}
			console.log("Quest completed!")
		}
		fn()
		console.log(`Spoofing video for ${applicationName}.`)
	} else if(taskName === "PLAY_ON_DESKTOP") {
		if(!isApp) {
			console.log("This no longer works in browser for non-video quests. Use the desktop app to complete the", applicationName, "quest!")
		}
		
		api.get({url: `/applications/public?application_ids=${applicationId}`}).then(res => {
			const appData = res.body[0]
			const exeName = appData.executables.find(x => x.os === "win32").name.replace(">","")
			
			const fakeGame = {
				cmdLine: `C:\\Program Files\\${appData.name}\\${exeName}`,
				exeName,
				exePath: `c:/program files/${appData.name.toLowerCase()}/${exeName}`,
				hidden: false,
				isLauncher: false,
				id: applicationId,
				name: appData.name,
				pid: pid,
				pidPath: [pid],
				processName: appData.name,
				start: Date.now(),
			}
			const realGames = RunningGameStore.getRunningGames()
			const fakeGames = [fakeGame]
			const realGetRunningGames = RunningGameStore.getRunningGames
			const realGetGameForPID = RunningGameStore.getGameForPID
			RunningGameStore.getRunningGames = () => fakeGames
			RunningGameStore.getGameForPID = (pid) => fakeGames.find(x => x.pid === pid)
			FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: realGames, added: [fakeGame], games: fakeGames})
			
			let fn = data => {
				let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value)
				console.log(`Quest progress: ${progress}/${secondsNeeded}`)
				
				if(progress >= secondsNeeded) {
					console.log("Quest completed!")
					
					RunningGameStore.getRunningGames = realGetRunningGames
					RunningGameStore.getGameForPID = realGetGameForPID
					FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: [fakeGame], added: [], games: []})
					FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
				}
			}
			FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
			
			console.log(`Spoofed your game to ${applicationName}. Wait for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
		})
	} else if(taskName === "STREAM_ON_DESKTOP") {
		if(!isApp) {
			console.log("This no longer works in browser for non-video quests. Use the desktop app to complete the", applicationName, "quest!")
		}
		
		let realFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata
		ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({
			id: applicationId,
			pid,
			sourceName: null
		})
		
		let fn = data => {
			let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.STREAM_ON_DESKTOP.value)
			console.log(`Quest progress: ${progress}/${secondsNeeded}`)
			
			if(progress >= secondsNeeded) {
				console.log("Quest completed!")
				
				ApplicationStreamingStore.getStreamerActiveStreamMetadata = realFunc
				FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
			}
		}
		FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
		
		console.log(`Spoofed your stream to ${applicationName}. Stream any window in vc for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
		console.log("Remember that you need at least 1 other person to be in the vc!")
	} else if(taskName === "PLAY_ACTIVITY") {
		const channelId = ChannelStore.getSortedPrivateChannels()[0]?.id ?? Object.values(GuildChannelStore.getAllGuilds()).find(x => x != null && x.VOCAL.length > 0).VOCAL[0].channel.id
		const streamKey = `call:${channelId}:1`
		
		let fn = async () => {
			console.log("Completing quest", applicationName, "-", quest.config.messages.questName)
			
			while(true) {
				const res = await api.post({url: `/quests/${quest.id}/heartbeat`, body: {stream_key: streamKey, terminal: false}})
				const progress = res.body.progress.PLAY_ACTIVITY.value
				console.log(`Quest progress: ${progress}/${secondsNeeded}`)
				
				await new Promise(resolve => setTimeout(resolve, 20 * 1000))
				
				if(progress >= secondsNeeded) {
					await api.post({url: `/quests/${quest.id}/heartbeat`, body: {stream_key: streamKey, terminal: true}})
					break
				}
			}
			
			console.log("Quest completed!")
		}
		fn()
	}
}
```
</details>

5. Follow the printed instructions depending on what type of quest you have
    - If your quest says to ("stream" and "play") the game, you can just wait and do nothing
7. Wait for 15 minutes
8. You can now claim the reward in User Settings -> Gift Inventory!

You can track the progress by looking at the `Quest progress:` prints in the Console tab, or by reopening the Gift Inventory tab in settings.

## FAQ

**Q: Ctrl + Shift + I doesn't work**

A: download the [ptb client](https://discord.com/api/downloads/distributions/app/installers/latest?channel=ptb&platform=win&arch=x64) to enable DevTools on stable



## Complete Discord Quest (PER)
 
> [!NOTE]
> Roye noskhe web ghabel estefade nist!
> 


> [!WARNING]
> Roye 2 no quest kar mikone ("stream" and "play")! Be dastoroamal tavajoh konid!
>

How to use this script:
1. Accept konid quest ro az bakhsh User Settings -> Gift Inventory
2. Press <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>I</kbd> to open DevTools
3. berid be bakhsh `Console`
4. Paste konid code ke ar bakhsh payin hast:
<details>
	<summary>Click to expand</summary>
	
```js
delete window.$;
let wpRequire = webpackChunkdiscord_app.push([[Symbol()], {}, r => r]);
webpackChunkdiscord_app.pop();

let ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getStreamerActiveStreamMetadata).exports.Z;
let RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.ZP?.getRunningGames).exports.ZP;
let QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getQuest).exports.Z;
let ChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getAllThreadsForParent).exports.Z;
let GuildChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.ZP?.getSFWDefaultChannel).exports.ZP;
let FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.flushWaitQueue).exports.Z;
let api = Object.values(wpRequire.c).find(x => x?.exports?.tn?.get).exports.tn;

let quest = [...QuestsStore.quests.values()].find(x => x.id !== "1248385850622869556" && x.userStatus?.enrolledAt && !x.userStatus?.completedAt && new Date(x.config.expiresAt).getTime() > Date.now())
let isApp = typeof DiscordNative !== "undefined"
if(!quest) {
	console.log("You don't have any uncompleted quests!")
} else {
	const pid = Math.floor(Math.random() * 30000) + 1000
	
	const applicationId = quest.config.application.id
	const applicationName = quest.config.application.name
	const taskName = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"].find(x => quest.config.taskConfig.tasks[x] != null)
	const secondsNeeded = quest.config.taskConfig.tasks[taskName].target
	let secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0

	if(taskName === "WATCH_VIDEO" || taskName === "WATCH_VIDEO_ON_MOBILE") {
		const maxFuture = 10, speed = 7, interval = 1
		const enrolledAt = new Date(quest.userStatus.enrolledAt).getTime()
		let fn = async () => {			
			while(true) {
				const maxAllowed = Math.floor((Date.now() - enrolledAt)/1000) + maxFuture
				const diff = maxAllowed - secondsDone
				const timestamp = secondsDone + speed
				if(diff >= speed) {
					await api.post({url: `/quests/${quest.id}/video-progress`, body: {timestamp: Math.min(secondsNeeded, timestamp + Math.random())}})
					secondsDone = Math.min(secondsNeeded, timestamp)
				}
				
				if(timestamp >= secondsNeeded) {
					break
				}
				await new Promise(resolve => setTimeout(resolve, interval * 1000))
			}
			console.log("Quest completed!")
		}
		fn()
		console.log(`Spoofing video for ${applicationName}.`)
	} else if(taskName === "PLAY_ON_DESKTOP") {
		if(!isApp) {
			console.log("This no longer works in browser for non-video quests. Use the desktop app to complete the", applicationName, "quest!")
		}
		
		api.get({url: `/applications/public?application_ids=${applicationId}`}).then(res => {
			const appData = res.body[0]
			const exeName = appData.executables.find(x => x.os === "win32").name.replace(">","")
			
			const fakeGame = {
				cmdLine: `C:\\Program Files\\${appData.name}\\${exeName}`,
				exeName,
				exePath: `c:/program files/${appData.name.toLowerCase()}/${exeName}`,
				hidden: false,
				isLauncher: false,
				id: applicationId,
				name: appData.name,
				pid: pid,
				pidPath: [pid],
				processName: appData.name,
				start: Date.now(),
			}
			const realGames = RunningGameStore.getRunningGames()
			const fakeGames = [fakeGame]
			const realGetRunningGames = RunningGameStore.getRunningGames
			const realGetGameForPID = RunningGameStore.getGameForPID
			RunningGameStore.getRunningGames = () => fakeGames
			RunningGameStore.getGameForPID = (pid) => fakeGames.find(x => x.pid === pid)
			FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: realGames, added: [fakeGame], games: fakeGames})
			
			let fn = data => {
				let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value)
				console.log(`Quest progress: ${progress}/${secondsNeeded}`)
				
				if(progress >= secondsNeeded) {
					console.log("Quest completed!")
					
					RunningGameStore.getRunningGames = realGetRunningGames
					RunningGameStore.getGameForPID = realGetGameForPID
					FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: [fakeGame], added: [], games: []})
					FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
				}
			}
			FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
			
			console.log(`Spoofed your game to ${applicationName}. Wait for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
		})
	} else if(taskName === "STREAM_ON_DESKTOP") {
		if(!isApp) {
			console.log("This no longer works in browser for non-video quests. Use the desktop app to complete the", applicationName, "quest!")
		}
		
		let realFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata
		ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({
			id: applicationId,
			pid,
			sourceName: null
		})
		
		let fn = data => {
			let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.STREAM_ON_DESKTOP.value)
			console.log(`Quest progress: ${progress}/${secondsNeeded}`)
			
			if(progress >= secondsNeeded) {
				console.log("Quest completed!")
				
				ApplicationStreamingStore.getStreamerActiveStreamMetadata = realFunc
				FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
			}
		}
		FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
		
		console.log(`Spoofed your stream to ${applicationName}. Stream any window in vc for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
		console.log("Remember that you need at least 1 other person to be in the vc!")
	} else if(taskName === "PLAY_ACTIVITY") {
		const channelId = ChannelStore.getSortedPrivateChannels()[0]?.id ?? Object.values(GuildChannelStore.getAllGuilds()).find(x => x != null && x.VOCAL.length > 0).VOCAL[0].channel.id
		const streamKey = `call:${channelId}:1`
		
		let fn = async () => {
			console.log("Completing quest", applicationName, "-", quest.config.messages.questName)
			
			while(true) {
				const res = await api.post({url: `/quests/${quest.id}/heartbeat`, body: {stream_key: streamKey, terminal: false}})
				const progress = res.body.progress.PLAY_ACTIVITY.value
				console.log(`Quest progress: ${progress}/${secondsNeeded}`)
				
				await new Promise(resolve => setTimeout(resolve, 20 * 1000))
				
				if(progress >= secondsNeeded) {
					await api.post({url: `/quests/${quest.id}/heartbeat`, body: {stream_key: streamKey, terminal: true}})
					break
				}
			}
			
			console.log("Quest completed!")
		}
		fn()
	}
}
```
</details>

5. Follow the printed instructions depending on what type of quest you have
    - Agar quest gofte ("stream" and "play") the game, faghat lazem hast ke sabr konid    
7. Wait for 15 minutes
8. Mitonid reward ro az bakhsh User Settings -> Gift Inventory!

You can track the progress by looking at the `Quest progress:` prints in the Console tab, or by reopening the Gift Inventory tab in settings.

## FAQ

**Q: Ctrl + Shift + I Kar nemikone**

A: In file ro download konid [ptb client](https://discord.com/api/downloads/distributions/app/installers/latest?channel=ptb&platform=win&arch=x64) ta bakhsh Dev tools faal beshe




