---
layout: post
authors: [elke_heymans]
title: 'Frontend Love Eindhoven 2019'
image: /img/frontend-love-eindhoven-2019/frontendloveeindhoven.png
tags: [Conference, Vue.js, React, Angular, Web Components, PWA]
category: Conference
comments: true
---

# Frontend Love Eindhoven 2019

On the 22nd of November, 8 of us went on a small roadtrip to Eindhoven to visit the inaugural Frontend Love Eindhoven edition.
With a total of 8 talks scheduled in the afternoon, it promised to be a very busy time.
The subjects of the talks were quite diverse: ranging from tips to improve your frontend components to sharing experiences with managing an open-source project.
In this blogpost we'll share some key things we've learned from these talks.

# The talks

## One component a day keeps the doctor away

By [Lucien Immink](https://twitter.com/lucienimmink){:target="_blank" rel="noopener noreferrer"}

The first talk of the day was given by Lucien who talked about web components.
The best way to define web components currently is by using `customElements`.
In the example he gave, he used `customElements` in combination with [lit-element](https://github.com/Polymer/lit-element){:target="_blank" rel="noopener noreferrer"}.
`lit-element` is a base class that can be reused to create web components as it provides you with the necessary options to define styles, properties, a render method and more.

```ts
class AlbumArt extends LitElement {
	static get properties() {
		return {
			artist: { type: String },
			// ... more properties ...
		};
	}
	static get styles() {
		return css`
			img {
				width: 100%;
				height: 100%;
			}
			p {
				margin: 0;
			}
		`;
	}
	constructor() {
		super();
		// set default properties, fetch data, ...
	}
	render() {
		return html`Insert HTML here`
	}
}
customElements.define("album-art", AlbumArt);
```

After we have defined the web component `AlbumArt` like this, we can use `<album-art></album-art>` wherever we want.

He also showed us the package [web-component-analyzer](https://www.npmjs.com/package/web-component-analyzer){:target="_blank" rel="noopener noreferrer"} to be able to document our web components.

[Watch Luciens's talk online](https://youtu.be/vNvuto5peM4){:target="_blank" rel="noopener noreferrer"}, [source code on GitHub](https://github.com/lucienimmink/album-art-webcomponent){:target="_blank" rel="noopener noreferrer"}

## The Evolution of Modern Web and Homo Frontendalis

By [Pooya Parsa](https://twitter.com/_pi0_){:target="_blank" rel="noopener noreferrer"}

[Watch Pooya's talk online](https://youtu.be/GyUqvz4rlrE){:target="_blank" rel="noopener noreferrer"}

## Advanced Vue.js features and patterns in the enterprise

By [Israel Roldan](https://twitter.com/isro_me){:target="_blank" rel="noopener noreferrer"}

Israel gave us an overview of 10 awesome built-in features that are used in enterprise Vue.js apps.

* vue-property-decorator
* Named scopes and scoped slots
* Functional components
* Global registration
* Built-in and custom directives
* Dynamic components
* Async components
* Controlled components
* Renderless components
* Dependency injection

[Watch Israel's talk online](https://youtu.be/q_VsYXiaBg8){:target="_blank" rel="noopener noreferrer"}

## Building Test Strategy for Vue.js application

By Anastasiia Dragich

No talk online?

## WebGL Demo with three.js

By [Colin van Eenige](https://twitter.com/cvaneenige){:target="_blank" rel="noopener noreferrer"}

Colin showed us the power of WebGL with three.js.

During his talk he made the statement:

> If you have a solid base structure, you're 80% of the way there

After which he showed how he created the demo.
He started up with setting up the base by having a loop that uses `requestAnimationFrame()` which tells the browser that you wish to perform an animation and requests that the browser calls a specified function to update an animation before the next repaint.
To draw something at every loop, he created a mesh by defining a geometry and a maeterial.
This mesh was to be repeated numerous times on a canvas by using [three.js's InstancedMesh](https://threejs.org/docs/#api/en/objects/InstancedMesh){:target="_blank" rel="noopener noreferrer"}.
After that he only had to define how the mesh had to move and how lights & rainbow colors would impact the visualisation.

This all culminated in a [demo application](https://exploring-webgl.app/){:target="_blank" rel="noopener noreferrer"} for which he added lots of parameters to show how powerful WebGL with three.js is.

[Watch Colin's talk online](https://youtu.be/Fi-h-TedHwc){:target="_blank" rel="noopener noreferrer"}, [WebGL with three.js demo](https://exploring-webgl.app/){:target="_blank" rel="noopener noreferrer"}

## Mistakes I made building React Async

By [Gert Hengeveld](https://twitter.com/GHengeveld){:target="_blank" rel="noopener noreferrer"}

Gert is one of the driving forces behind the [React Async package](https://www.npmjs.com/package/react-async){:target="_blank" rel="noopener noreferrer"}.
During its development, he learned some key things:

* Breaking builds: you can unpublish a package within 72 hours of publish
* Forgetting integrations
* It's hard to keep typing definitions in sync: easy to solve, migrate to TypeScript
* Not writing about it: it's not enough to write a good package, you also need to do the marketing to gain some momentum
* Going at it alone: try to trust others and ask help to build your package

[Watch Gert's talk online](https://youtu.be/wyBZtQO0xo8){:target="_blank" rel="noopener noreferrer"}, [React Async package](https://www.npmjs.com/package/react-async){:target="_blank" rel="noopener noreferrer"}

## Angular & ElasticSearch: combined forces

By Franciska van Maurik

[Watch Franciska's talk online](https://youtu.be/Nnmh6piw4UQ){:target="_blank" rel="noopener noreferrer"}

## Native-like PWAs in Web Components live-code

By [Jad Joubran](https://twitter.com/JoubranJad){:target="_blank" rel="noopener noreferrer"}

[Watch Jad's talk online](https://youtu.be/59hY-QOxXxo){:target="_blank" rel="noopener noreferrer"}

# Conclusion

Conclusion comes here

Conference twitter: https://twitter.com/Frontend_Love
Organisers twitter: https://twitter.com/passionpeopleNL & https://twitter.com/isaaceindhoven
