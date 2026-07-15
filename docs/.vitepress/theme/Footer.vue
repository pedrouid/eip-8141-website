<template>
  <footer class="custom-footer">
    <div class="footer-container">
      <div class="footer-col footer-title">
        <h4>EIP-8141 Frame Transaction</h4>
      </div>
      <div class="footer-col">
        <h4>Spec</h4>
        <ul>
          <li><a href="/current-spec">Current Spec</a></li>
          <li><a href="/feedback-evolution">Feedback Evolution</a></li>
          <li><a href="/original-spec">Original Spec</a></li>
          <li><a href="/merged-changes">Merged Changes</a></li>
          <li><a href="/original-vs-latest">Original vs Latest</a></li>
        </ul>
      </div>
      <div class="footer-col">
        <h4>Topics</h4>
        <ul>
          <li><a href="/eoa-support">EOA Support</a></li>
          <li><a href="/pq-roadmap">PQ Roadmap</a></li>
          <li><a href="/mempool-strategy">Mempool Strategy</a></li>
          <li><a href="/vops-compatibility">VOPS Compatibility</a></li>
          <li><a href="/developer-tooling">Developer Tooling</a></li>
          <li><a href="/competing-standards">Competing Standards</a></li>
        </ul>
      </div>
      <div class="footer-col">
        <h4>Resources</h4>
        <ul>
          <li>
            <a
              href="https://github.com/ethereum/EIPs/blob/master/EIPS/eip-8141.md"
              target="_blank"
              rel="noopener"
              >EIP-8141 Spec</a
            >
          </li>
          <li>
            <a
              href="https://github.com/ethereum/EIPs/pulls?q=is%3Apr+8141"
              target="_blank"
              rel="noopener"
              >All Related PRs</a
            >
          </li>
          <li>
            <a
              href="https://ethereum-magicians.org/t/frame-transaction/27617"
              target="_blank"
              rel="noopener"
              >EthMagicians Discussion</a
            >
          </li>
          <li>
            <a
              href="https://demo.eip-8141.ethrex.xyz/"
              target="_blank"
              rel="noopener"
              >Live Demo</a
            >
          </li>
          <li>
            <a
              href="https://github.com/pedrouid/eip-8141-website"
              target="_blank"
              rel="noopener"
              >GitHub Repository</a
            >
          </li>
        </ul>
      </div>
    </div>
  </footer>
</template>

<style scoped>
.custom-footer {
  border-top: 1px solid var(--vp-c-divider);
  padding: 32px 32px;
}

.footer-container {
  max-width: calc(var(--vp-layout-max-width) - 64px);
  margin: 0 auto;
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 32px;
}

.footer-col h4 {
  font-size: 14px;
  font-weight: 600;
  color: var(--vp-c-text-1);
  margin-bottom: 12px;
}

.footer-col ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.footer-col li {
  margin-bottom: 8px;
}

.footer-col a {
  font-size: 13px;
  color: var(--vp-c-text-2);
  text-decoration: none;
  transition: color 0.2s;
}

.footer-col a:hover {
  color: var(--vp-c-brand-1);
}

@media (max-width: 1024px) {
  .footer-title {
    display: none;
  }

  .footer-container {
    grid-template-columns: repeat(2, 1fr);
  }
}

@media (max-width: 768px) {
  .custom-footer {
    padding: 32px 24px;
  }

  .footer-container {
    grid-template-columns: 1fr;
    gap: 24px;
  }
}
</style>
