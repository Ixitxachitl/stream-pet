# Stream Pet

An OBS Browser Source overlay with a little voxel critter that lives along the bottom of your stream and
looks after itself. It hatches, grows up and evolves according to how its life on your stream went.

Live at [pet.ixitxachitl.com](https://pet.ixitxachitl.com) once it's deployed.

## Try it

```sh
python -m http.server 8000
# then open http://localhost:8000/?debug=1&ball=1
```

Hook it up to chat with `?channel=yourtwitch` (or `kick=`, `tiktok=1` for TikFinity, `youtube=` + `ytkey=`).
Chat is optional: it makes the pet more sociable and changes how it grows up, but it looks after itself
either way. Subs drop cake, bits rain coins and raids bring confetti.

## How it works

- **It looks after itself.** It gets hungry, sleepy, bored and messy, and deals with it: food drops from
  a dispenser, it plays with its ball, sleeps through its night hours and a tiny broom sweeps up. Nobody
  has to press anything, and time away (the source hidden, a week off stream) never leaves it in a bad
  way.
- **It grows up with the channel.** Egg, baby, child, teen, adult and elder, counted in hours while the
  source is visible. The teen form comes from its strongest trait; the adult is one of 672 forms (plus
  rare ones) picked from how its life went: what it did, when you stream and what happened in chat.
- **Evolution is a surprise.** Nothing on stream shows where it's heading. The same kind of life gives
  the same form on any channel, so viewers can compare pets.
- **Families.** An elder eventually retires: it lays an egg, waves goodbye and heads off. Each egg keeps
  some of its parent's colouring, and over the generations a family can grow walking legs (in place of
  hopping), dangly arms and finer voxels. Playful parents pass on stronger legs, social ones pass on
  arms. The [walk lab](walk-lab.html) shows how that looks.

It's one `index.html` with no build step, built with three.js and Rapier physics. Settings are in the
gear panel (via OBS Interact) and every setting also works as a URL parameter.

For testing: `debug=1`, `demo=1` (fake chat that doesn't count), `stage=` / `form=` for a throwaway pet,
`gallery=1` to see every form (spoilers), `legs=` / `arms=` (0–1) for a pet whose family has grown
walking legs and arms, `detail=` (1–3) for one built from finer voxels, `speed=` and `agerate=` to
fast-forward. The full list is at the top of [index.html](index.html).

## Not done yet

- Testing in real OBS (hardware acceleration on and off, long sessions).
- Deploying to pet.ixitxachitl.com.
- A review pass over every form in the gallery (`?gallery=1`, also with `detail=3`).

## Credits

Built with help from [Claude Code](https://claude.com/claude-code). Uses [three.js](https://threejs.org) and
[Rapier](https://rapier.rs).

## License

GPL-3.0 — see [LICENSE](LICENSE).
