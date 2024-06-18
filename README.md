# Adi's notes

Build & serve the website locally:

```bash
npm ci
npx quartz build --serve

npx quartz sync
```

## Obsidian nested vaults

To link the content directory to another Obsidian Vault, under Windows you have to make a [junction directory](https://help.obsidian.md/Files+and+folders/Symbolic+links+and+junctions):

```powershell
mklink /J "C:\Users\..\public_notes" "C:\Users\..\content"
```

## Acknowledgements

- 🔗 Edited and managed in [Obsidian](https://obsidian.md/)
- 🔗 Based off of [Quartz 4](https://quartz.jzhao.xyz/)
- Contains ["burn out" to Markdown from Dataview Plugin](https://joschua.io/posts/2023/09/01/obsidian-publish-dataview/)
