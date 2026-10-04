# earth-network-docs

Documentation for the earth network, published at **https://docs.erth.network**.

    docs/                     user guides: what the chain is and how to use it
    docs/run-a-node/          node operators: joining, the consensus key, upgrades
    docs/operations/          runbooks for things that go wrong
    docs/technical/           how the core pieces work: personhood, the fund streams

## Local

    npm ci
    npm start          # live reload on :3000
    npm run build      # what CI publishes

`onBrokenLinks: "throw"` — a page that lies about where something is, is worse
than a missing page, so a broken link fails the build rather than shipping.

## Publishing

Pushing to `main` builds and deploys to GitHub Pages. The custom domain comes
from `static/CNAME`; the Docusaurus config reads that file and switches `url`
and `baseUrl` on its presence, so the two cannot disagree.

The chain itself lives at
[zenopie/earth-network-chain](https://github.com/zenopie/earth-network-chain) —
node software, genesis, and the image. Nothing here is needed to run a node;
this is only how it is explained.
