help:
	@echo "Available targets:"
	@echo "  run        - Bootstrap the full environment (install tools, provision cluster)"
	@echo "  down       - Destroy the cluster and all resources"
	@echo "  push       - Bump patch version, tag, and push to trigger CI"
	@echo "  tools      - Install necessary tools only"
	@echo "  tofu       - Initialize OpenTofu"
	@echo "  apply      - Apply OpenTofu configuration"
	@echo "  fix-egress - Repair nested-Docker egress (Codespaces) and verify nodes"
	@echo "  fix-docker-acl - Clear the /tmp default ACL that breaks non-root images"

run:
	@bash scripts/setup.sh

tools:
	@curl -fsSL https://get.opentofu.org/install-opentofu.sh | sh -s -- --install-method standalone
	@curl -sS https://webi.sh/k9s | bash
	@ARCH=$$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/'); \
	  OS=$$(uname -s | tr '[:upper:]' '[:lower:]'); \
	  curl -fsSLo /tmp/kind "https://kind.sigs.k8s.io/dl/v0.33.0/kind-$$OS-$$ARCH" && \
	  sudo install -m 0755 /tmp/kind /usr/local/bin/kind && rm -f /tmp/kind

fix-egress:
	@bash scripts/fix-egress.sh
	@bash scripts/fix-egress.sh verify abox

fix-docker-acl:
	@bash scripts/fix-docker-acl.sh
	@bash scripts/fix-docker-acl.sh verify abox

tofu:
	@cd bootstrap && tofu init

apply:
	@cd bootstrap && tofu apply -auto-approve

down:
	@cd bootstrap && tofu destroy -auto-approve

push:
	@git fetch origin --tags --force
	@set -eu; \
	  latest_tag=$$(git tag --list 'v*' | grep -E '^v[0-9]+\.[0-9]+\.[0-9]+$$' | sort -V | tail -n 1 || true); \
	  if [ -z "$$latest_tag" ]; then latest_tag=v0.0.0; fi; \
	  version=$${latest_tag#v}; \
	  major=$${version%%.*}; \
	  remainder=$${version#*.}; \
	  minor=$${remainder%%.*}; \
	  patch=$${remainder#*.}; \
	  new_tag="v$$major.$$minor.$$((patch + 1))"; \
	  branch=$$(git branch --show-current); \
	  if [ -z "$$branch" ]; then echo "Cannot publish from detached HEAD" >&2; exit 1; fi; \
	  git push origin "HEAD:refs/heads/$$branch" "HEAD:refs/tags/$$new_tag"; \
	  echo "Pushed branch $$branch and tag $$new_tag"
