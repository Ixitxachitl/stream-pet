# Stream Pet

An OBS Browser Source overlay with a little voxel critter that lives along the bottom of your stream and
looks after itself. It hatches, grows up and evolves according to how its life on your stream went.

Work in progress, see [PLAN.md](PLAN.md). Live at [pet.ixitxachitl.com](https://pet.ixitxachitl.com) once it's ready.

## Try it

```sh
python -m http.server 8000
# then open http://localhost:8000/?debug=1&ball=1
```

Useful URL parameters for now: `stage` (egg, baby, child, adult), `seed`, `size`, `ground`, `xmin`/`xmax`,
`ball=1`, `debug=1`, `quality=low`. The full list is at the top of [index.html](index.html).

## Credits

Built with help from [Claude Code](https://claude.com/claude-code). Uses [three.js](https://threejs.org) and
[Rapier](https://rapier.rs).

## License

GPL-3.0 — see [LICENSE](LICENSE).
