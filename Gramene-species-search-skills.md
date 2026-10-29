I saved the method as a Claude Code skill called `gramene-species-search`, at `/home/weix/.claude/skills/gramene-species-search/SKILL.md`.

It's picked up automatically in future Claude Code sessions on this server. A request like "add a GrameneWheat Search button for Triticum_aestivum IWGSC" will use it, or you can type `/gramene-species-search`.

What it covers:
- **The four nodes so far:** Gramene Search, Sorghumbase Search, GrameneOryza and GrameneMaize Search, each with its species, assembly and link format.
- **Finding the species and assembly:** the command that reads a species' exact assembly name from the site config, and a reminder to keep related genomes like `Oryza_sativa_ir64` from getting the node.
- **Checking the link:** test the external link with a real gene ID, with and without the `/` before `?idList=`, because SorghumBase returns 404 with the slash.
- **The code:** a template for the new page and the `Gene.pm` block.
- **The mistake to avoid:** don't wrap `create_node` in `->after()`. That is what broke the Sorghum gene pages.
- **Restart and testing:** you restart Apache yourself, then it checks the page with `curl` and reads `logs/error.log` if a page fails.
- **Committing:** commit to `gramene-ensembl-70`, push over SSH, and check that no passwords are in the diff.

The GrameneOryza and GrameneMaize nodes still need an Apache restart. Tell me when it's back up and I'll check both pages.