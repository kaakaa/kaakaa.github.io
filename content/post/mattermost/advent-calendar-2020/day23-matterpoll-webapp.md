---
title: "[Mattermost Integrations] Matterpoll Plugin (Webapp)"
date: 2020-12-23T00:00:00+09:00
draft: false
toc: true
tags: ["mattermost", "integration", "plugin", "matterpoll"]
---

Mattermost記事まとめ: https://blog.kaakaa.dev/tags/mattermost/

## 本記事について

[Mattermostの統合機能アドベントカレンダー](https://qiita.com/advent-calendar/2020/mattermost-integrations)の第23日目の記事です。

以前から開発に参加している[Matterpoll](https://github.com/matterpoll/matterpoll)がMattermost公式のPlugin MarketplaceでCommunity Pluginとして公開されました。

昨日の記事ではMatterpoll PluginのServer側の実装について紹介しましたが、本日の記事ではMatterpoll PluginのWebapp側の機能について紹介します。

## Webapp側の処理

Webapp側では、ユーザー毎に投票の見え方を変えるための処理を実装しています。

#### 独自のPostTypeによる投票済み回答のハイライト

![matterpoll-posttype.dio.png](https://blog.kaakaa.dev/images/posts/advent-calendar-2020/day23/matterpoll-posttype.dio.png)

Mattermosで投稿したメッセージは、Mattermost上では [Post]([https://github.com/mattermost/mattermost-server/blob/master/model/post.go#L72](https://github.com/mattermost/mattermost-server/blob/master/model/post.go#L72)) 構造体のデータとして扱われています。そして、`Post`構造体の`Type` フィールドの値によって投稿の種別を判別しています。`Post.Type`はデフォルトでは大きく分けて二つの種類があり、一つは通常ユーザーが投稿するメッセージ（ `Post.Type` が空文字（`POST_DEFAULT`）。もう一つは、チャンネル参加時などにシステムによって投稿されるメッセージ（`Post.Type` は [`system_` で始まる]([https://github.com/mattermost/mattermost-server/blob/master/model/post.go#L22](https://github.com/mattermost/mattermost-server/blob/master/model/post.go#L22))）です。

![2 つのPost.Typeの投稿](https://blog.kaakaa.dev/images/posts/advent-calendar-2020/day23/posttype-sample.png)

Mattermost上のメッセージの見た目は [Message Attachements]([https://docs.mattermost.com/developer/message-attachments.html](https://docs.mattermost.com/developer/message-attachments.html)) の機能を使う事である程度変更することはできますが、 Message AttachmentsではMattermostユーザー全員に対して同じ見た目しか提供できないため、ユーザー毎にメッセージの一部を変更したい場合などはプラグインで処理する必要があります。Matterpollの場合「自分が投票した回答がどれなのか表示したい」というようなユーザー毎に表示を変更する要求があったため、この部分をプラグインで処理しています。

プラグインで投稿の見た目を変えたい場合、まずServer側で投稿を作成する際に、独自の`Post.Type`を設定する必要があります。Matterpollではスラッシュコマンド `/poll` が実装された際に投票用の投稿  (`Post`) を作成するのですが、その際に `Post.Type`  に `custom_matterpoll` という値を設定しています。

```go:server/plugin/plugin.go
...
const (
  ...
	// MatterpollPostType is post_type of posts generated by Matterpoll
	MatterpollPostType = "custom_matterpoll"
)
...
```

[https://github.com/matterpoll/matterpoll/blob/45f095875a98fb1d4f3f166851c86f41b987493e/server/plugin/plugin.go#L53](https://github.com/matterpoll/matterpoll/blob/45f095875a98fb1d4f3f166851c86f41b987493e/server/plugin/plugin.go#L53)

```go:server/plugin/command.go
...
  post := &model.Post{
		UserId:    p.botUserID,
		ChannelId: args.ChannelId,
		RootId:    args.RootId,
		Type:      MatterpollPostType,
		Props: map[string]interface{}{
			"poll_id": newPoll.ID,
		},
	}
	model.ParseSlackAttachment(post, actions)

	rPost, appErr := p.API.CreatePost(post)
...
```

[https://github.com/matterpoll/matterpoll/blob/45f095875a98fb1d4f3f166851c86f41b987493e/server/plugin/command.go#L191](https://github.com/matterpoll/matterpoll/blob/45f095875a98fb1d4f3f166851c86f41b987493e/server/plugin/command.go#L191)

上記のように`Post.Type` にプラグイン独自の値を設定しましたが、Webapp側で`custom_matterpoll`という`Post.Type`に対する処理を追加しない限り、通常の投稿と同じように表示されてしまいます。Mattermostプラグインでは、WebApp プラグインAPIの[registerPostTypeComponent]([https://developers.mattermost.com/extend/plugins/webapp/reference/#registerPostTypeComponent](https://developers.mattermost.com/extend/plugins/webapp/reference/#registerPostTypeComponent)) を使い `Post.Type` に `custom_matterpoll` が設定された場合のレンダリング方法を指定しています。

[registerPostTypeComponent]([https://developers.mattermost.com/extend/plugins/webapp/reference/#registerPostTypeComponent](https://developers.mattermost.com/extend/plugins/webapp/reference/#registerPostTypeComponent))  は第一引数に`Post.Type` の文字列を指定し、第二引数にReactコンポーネントを指定します。Matterpollの場合は、Server側で `Post.Type` に `custom_matterpoll` を指定しているため、下記のように第一引数に `custom_matterpoll` を、第二引数に独自のReactコンポーネントを指定して [registerPostTypeComponent]([https://developers.mattermost.com/extend/plugins/webapp/reference/#registerPostTypeComponent](https://developers.mattermost.com/extend/plugins/webapp/reference/#registerPostTypeComponent)) を呼び出しています。

```jsx:webapp/src/actions/config.js
import PostType from 'components/post_type';
...

    if (data.experimentalui) {
        registeredComponentId = registry.registerPostTypeComponent('custom_matterpoll', PostType);
    } else {
       registry.unregisterPostTypeComponent(registeredComponentId);
       registeredComponentId = '';
    }
...
```

[https://github.com/matterpoll/matterpoll/blob/fa1f3aa8489deb24706937beac7d9976b0528ed7/webapp/src/actions/config.js#L11](https://github.com/matterpoll/matterpoll/blob/fa1f3aa8489deb24706937beac7d9976b0528ed7/webapp/src/actions/config.js#L11)

[registerPostTypeComponent]([https://developers.mattermost.com/extend/plugins/webapp/reference/#registerPostTypeComponent](https://developers.mattermost.com/extend/plugins/webapp/reference/#registerPostTypeComponent)) は戻り値として登録されたReactコンポーネントに対応するID文字列を返しますが、これは登録したコンポーネントを[unregisterPostTypeComponent]([https://developers.mattermost.com/extend/plugins/webapp/reference/#unregisterPostTypeComponent](https://developers.mattermost.com/extend/plugins/webapp/reference/#registerPostTypeComponent))を利用して解除する場合に必要なためstateに保持しています。

Matterpollでは、Webapp側の機能はまだExperimentalな機能として設定でOn/Offを切り替えられるようにしているため、設定変更によってコンポーネントを登録/解除できるようにしています。

このように、プラグイン独自に投稿の見た目を変えたい場合、自作のReactコンポーネントを作成する必要があります。この時、Mattermost本体が構築する投稿に関するDOMの一部だけ表示を変更するなどができると良いのですが、MattermostのReactコンポーネントは外部から参照できる形で公開されていないため、一から自分で作る必要がありなかなか難儀しています...。ただ、Reactコンポーネントを開発する際のユーティリティ関数がいくつか利用可能なため、この辺りを使っいます。

[https://developers.mattermost.com/extend/plugins/webapp/reference/#post-utils](https://developers.mattermost.com/extend/plugins/webapp/reference/#post-utils)

#### プラグイン独自のWebSocketイベント

![matterpoll-ws.dio.png](https://blog.kaakaa.dev/images/posts/advent-calendar-2020/day23/matterpoll-ws.dio.png)

MattermostのServerとWebappの間はWebSocketによりやり取りが行われています。

Webapp側でWebSocketイベントを受信した時の独自処理を登録するのが [registerWebSocketEventHandler]([https://developers.mattermost.com/extend/plugins/webapp/reference/#registerWebSocketEventHandler](https://developers.mattermost.com/extend/plugins/webapp/reference/#registerWebSocketEventHandler))メソッドです。このメソッドは、第一引数にWebSocketイベント名、第二引数にWebSocketイベントを受け取ったときに実行される処理を関数として登録します。

MattermostがデフォルトでPublishしているWebSocketイベントは[APIドキュメント]([https://api.mattermost.com/#tag/WebSocket](https://api.mattermost.com/#tag/WebSocket))に一覧でまとめられており、メッセージの投稿/削除/編集やチャンネルの作成/削除、参加/脱退など、ユーザー操作に対するイベントが一通り網羅されています。また、Server側からプラグインごとに独自のWebSocketイベントをPublishすることもでき、Matterpollでは、Matterpollプラグインの設定が変更された時と、ユーザーが投票に回答したときに独自のWebSocketイベントを送信しています。

プラグイン独自のWebSocketイベントをPublishするには、Server側でプラグインAPIの[PublishWebSocketEvent]([https://developers.mattermost.com/extend/plugins/server/reference/#API.PublishWebSocketEvent](https://developers.mattermost.com/extend/plugins/server/reference/#API.PublishWebSocketEvent))を実行するだけでよく、Matterpollでは下記のようなメソッドを作成し、投票データに変更があった時にこの処理を呼び出すことで、投票に関する最新のデータをWebappに送信しています。

```go:server/plugin/api.go
func (p *MatterpollPlugin) publishPollMetadata(poll *poll.Poll, userID string) {
	hasAdminPermission, appErr := p.HasAdminPermission(poll, userID)
	if appErr != nil {
		p.API.LogWarn("Failed to check admin permission", "userID", userID, "pollID", poll.ID, "error", appErr.Error())
		hasAdminPermission = false
	}
	metadata := poll.GetMetadata(userID, hasAdminPermission)
	p.API.PublishWebSocketEvent("has_voted", metadata.ToMap(), &model.WebsocketBroadcast{UserId: userID})
}
```

[https://github.com/matterpoll/matterpoll/blob/45f095875a98fb1d4f3f166851c86f41b987493e/server/plugin/api.go#L411](https://github.com/matterpoll/matterpoll/blob/45f095875a98fb1d4f3f166851c86f41b987493e/server/plugin/api.go#L411)

プラグインAPIである [PublishWebSocketEvent]([https://developers.mattermost.com/extend/plugins/server/reference/#API.PublishWebSocketEvent](https://developers.mattermost.com/extend/plugins/server/reference/#API.PublishWebSocketEvent))  は3つの引数を取り、第一引数でPublishするWebSocketイベントのイベント名を指定します。実際には、ここで指定されたイベント名の先頭に `custom_<pluginid>_` を付与したイベント名としてやり取りされるため、第1引数を "has_voted" としている今回の例では `custom_com.github.matterpoll.matterpoll_has_voted`というWebSocketイベント名となります。イベント名にプラグインIDが入っているため、プラグイン間のイベント名の競合については考慮する必要がありません。

第2引数にはPublishするデータを指定します。`map` 形式でvalueには `interface{}` 型が指定できますが、この値にはGoのプリミティブ型か [mattermost-server/model]([https://github.com/mattermost/mattermost-server/tree/master/model](https://github.com/mattermost/mattermost-server/tree/master/model)) しか指定できません。それ以外の値を指定してしまうと、実行時にエンコードエラーとなります。（Mattermostサーバーのログを確認しないと分からないためハマりがちです）

第3引数では、このWebSocketイベントをどのユーザーに対して送信するかを決定します。[model.WebsocketBroadcast]([https://pkg.go.dev/github.com/mattermost/mattermost-server/v5/model#WebsocketBroadcast](https://pkg.go.dev/github.com/mattermost/mattermost-server/v5/model#WebsocketBroadcast))の定義を見ると、ユーザー単位以外にもチャンネルやチーム単位で指定することもできそうです（試したことはないですが）。

次に、Server側からPublishされたWebSocketイベント `custome_com.github.matterpoll.matterpoll_has_voted` をWebapp側で処理するために、前述の [registerWebSocketEventHandler]([https://developers.mattermost.com/extend/plugins/webapp/reference/#registerWebSocketEventHandler](https://developers.mattermost.com/extend/plugins/webapp/reference/#registerWebSocketEventHandler)) を実行します。

```jsx:webapp/src/plugin.jsx
export default class MatterPollPlugin {
    async initialize(registry, store) {
        ...
        registry.registerWebSocketEventHandler(
            'custom_' + pluginId + '_has_voted',
            (message) => {
                store.dispatch(websocketHasVoted(message.data));
            },
        );
        ...
    }
```

[https://github.com/matterpoll/matterpoll/blob/0b797025cfbf319c43464fcf457d4cdfe5086188/webapp/src/plugin.jsx#L17](https://github.com/matterpoll/matterpoll/blob/0b797025cfbf319c43464fcf457d4cdfe5086188/webapp/src/plugin.jsx#L17)

上記のように、Webapp側のプラグイン起動時にWebSocketを受信した時のHandlerを登録しておくことで、Mattermost上の操作に応じてWebapp側の表示を変えるなどの処理を行えるようになります。

## さいごに

本日は、Mattermost上で投票を行えるようにするMatterpoll PluginのWebapp側の実装について紹介しました。
明日は、Matterpoll Plugin開発におけるテストやCIについて紹介します。