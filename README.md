# Stream Pet

An OBS Browser Source overlay with a little voxel critter that lives along the bottom of your stream and
looks after itself. It hatches, grows up and evolves according to how its life on your stream went.

Work in progress, see [PLAN.md](PLAN.md). Live at [pet.ixitxachitl.com](https://pet.ixitxachitl.com) once it's ready.

## Try it

```sh
python -m http.server 8000
# then open http://localhost:8000/?debug=1&ball=1
```

Hook it up to chat with `?channel=yourtwitch` (or `kick=`, `tiktok=1` for TikFinity, `youtube=` + `ytkey=`).
Chat is optional: it makes the pet more sociable and changes how it grows up, but it looks after itself
either way. Subs drop cake, bits rain coins and raids bring confetti.

For testing: `debug=1`, `demo=1` (fake chat that doesn't count), `stage=` / `form=` for a throwaway pet,
`gallery=1` to see every form (spoilers), `legs=` / `arms=` (0–1) for a pet whose family has grown
walking legs and arms, `detail=` (1–3) for one built from finer voxels, `speed=` and `agerate=` to
fast-forward. The full list is at
the top of [index.html](index.html).

## Credits

Built with help from [Claude Code](https://claude.com/claude-code). Uses [three.js](https://threejs.org) and
[Rapier](https://rapier.rs).

## License

GPL-3.0 — see [LICENSE](LICENSE).
