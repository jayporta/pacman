# Pac-Man

I wanted to get a sense of what it could be like to work in a legacy codebase using an AI assistant, so I searched GitHub for a repo from 2010-2018 that has not been modified yet.

## The plan:

1. Run a local code review using Claude Code CLI Opus model.
2. Plan out a strategy if there are things to fix or clean up.
3. Gradually update this to Web Components and TypeScript while considering new features or changes that can be implemented while I'm at it. No framework this time, which keeps the DOM work close to the platform.
4. Play Pac-Man.
5. Look for something older.
6. Repeat... probably without Pac-Man.

Progress is tracked task by task on the [migration board](https://github.com/users/jayporta/projects/3). One pull request per task, and nothing lands without a review.

## Running it

```bash
python3 -m http.server 8000   # then open http://localhost:8000
```

Serve it over HTTP instead of opening the file directly. The sound library builds its audio objects from relative paths the moment the page parses, and `file://` blocks that in most browsers.

## Credits

A fork of [luciopanepinto/pacman](https://github.com/luciopanepinto/pacman), written by Lucio Panepinto in 2015. All the original game design and canvas work is theirs.

Copyright (C) 2015 Lucio Panepinto. Modified by Jay Porta from 2026 onward.

## License

[GNU General Public License v3.0](LICENSE).

This fork stays GPL-3.0 and keeps the original copyright notice. Tracking, personal branding, and ad links from the upstream project have been removed, but attribution has not and will not be.

### Original README content:

> Pac-Man game written in HTML5 + CSS3 + jQuery with Canvas. This WebApp is a Responsive Web Design (RWD) website.
