# Hybrid Server Launcher

Java 25 executable launcher that starts one public proxy and one or more private game servers. The first run enables only the proxy and lobby. Maven creates the runnable artifact at `target/server.jar`.

## Build

Use JDK 25 and Maven 3.9 or newer:

```sh
mvn clean package
java -jar target/server.jar
```

The first run creates `config/hybrid.properties`, `run/` directories, plugin directories, and EULA files for vanilla/modded game servers. Review Mojang's EULA and set `eula=true` in each applicable enabled server's `run/<name>/eula.txt`; the `limbo` backend type skips the EULA check. Place the selected server jars under `server-libs/`.

## Proxy and lobby

Set `proxy.type` to `velocity`, `bungeecord`, or `waterfall`. Set `lobby.type` to `paper`, `purpur`, `spigot`, `folia`, `pufferfish`, `leaf`, `leaves`, `fabric`, `forge`, `neoforge`, `arclight`, `mohist`, or `limbo`.

```properties
proxy.type=velocity
proxy.jar=server-libs/velocity.jar
proxy.bind=0.0.0.0:25565
proxy.memory=1G
proxy.java=java
proxy.online-mode=true
proxy.velocity-forwarding-mode=auto

lobby.type=paper
lobby.jar=server-libs/paper.jar
lobby.port=25566
lobby.memory=4G
lobby.java=java
lobby.launch-args=auto
lobby.minecraft-version=1.21.5
# Optional matching Forge/NeoForge libraries archive:
# lobby.libs=libs/forge-1.21.1-libraries.zip

plugin-servers=
```

`launch-args=auto` detects the software's normal launch profile: standard server
JARs use `-jar server.jar`, Fabric uses its server launcher, and Forge/NeoForge
use generated argument files or Forge server JARs. If a Forge/NeoForge server
installer is supplied as the configured JAR, the launcher runs its
`--installServer` step first. Existing custom arguments remain supported.
Set the JAR path to `none` only when the required server files already exist
in that server's `run/<name>/` directory.

## Additional game servers

Add server names to the comma-separated `plugin-servers` property to enable them. For each name, configure `plugin.<name>.type`, `.jar`, `.memory`, `.java`, and optionally `.port`. The type can be any supported lobby type, so multiple servers may use the same software or different software. JAR paths default to `server-libs/<name>.jar`; Java defaults to `lobby.java`; memory defaults to `2G`. If a port is omitted, the launcher assigns the next port after the lobby and preceding game server.

For example, with lobby port `25567`, `skyblock` gets port `25568` unless explicitly configured otherwise:

```properties
plugin-servers=skyblock, survival
plugin.skyblock.type=paper
plugin.skyblock.jar=server-libs/skyblock-paper.jar
plugin.skyblock.memory=3G
plugin.skyblock.port=25568
plugin.survival.type=folia
plugin.survival.jar=server-libs/survival-folia.jar
plugin.survival.memory=4G
```

Every game server binds to `127.0.0.1`; its proxy entry is generated automatically. Each runs in `run/<name>/`. Put that server's plugins in `plugins/<name>/`; proxy plugins go in `plugins/proxy/`. The lobby's plugin directory is `plugins/lobby/`.

Console commands without a prefix are sent to every game server, including the lobby. Use `psw <name> <command>` to target only one, for example `psw skyblock say Hello`. Proxy processes display their logs but do not receive console commands.

The launcher itself requires Java 25+. Proxy/backend Java commands are checked independently and must be Java 8+; choose a runtime compatible with each exact server build (older Minecraft/Forge releases may need older Java, while newer builds may need newer Java). Generated proxy settings are updated to include enabled backends while other existing config entries are preserved. For Velocity, `proxy.velocity-forwarding-mode` accepts `auto`, `modern`, `legacy`, or `none`. `auto` selects modern for Fabric/Forge/NeoForge, legacy for Spigot, and otherwise respects an existing `player-info-forwarding-mode` in `run/proxy/velocity.toml` (defaulting to modern for a new config). Arclight and Mohist are no longer automatically forced to legacy: choose the mode with `proxy.velocity-forwarding-mode`, or retain a valid mode already in `velocity.toml`. Arclight modern mode updates `arclight.conf` `velocity.enable` and the shared secret; Mohist modern mode updates `mohist-config/mohist.yml` `velocity.enabled`, `velocity.onlineMode`, and `velocity.secret`. Limbo's `server.properties` is configured with the selected proxy's forwarding mode and secret, bound to localhost, and assigned its launcher port. Legacy mode enables Bungee forwarding and disables native modern forwarding; none disables forwarding. A valid mode manually set in `velocity.toml` is honored when the property is `auto`; other TOML settings are left intact. BungeeCord and Waterfall use Bungee forwarding.
Paper-family forwarding config follows the backend's Minecraft version: Paper 1.19.4+ uses `config/paper-global.yml`, while older versions use `paper.yml` and `settings.velocity-support`. The launcher updates only the forwarding keys and keeps the Velocity secret in sync. Set `lobby.minecraft-version` or `plugin.<name>.minecraft-version` when the version cannot be detected from the server JAR or existing Paper config.
Set `lobby.java` / `plugin.<name>.java` to an installed Java executable, such as `java` when it is on `PATH`, or to the actual executable path provided by the host. Example paths are placeholders and must not be copied literally. If the host only provides Java 25, an older Forge backend that requires Java 8 will need a Java 8 runtime enabled/provided by the host. BungeeCord and Waterfall use Bungee forwarding.
For backend startup only, the launcher first tries the configured Java. If the process exits before opening its listener and prints a recognized Java-compatibility error (Paper's maximum-supported-Java message, an unsupported class-file version, or the known old-Forge `sun.security.util` error), it downloads a matching Eclipse Temurin JRE into `runtimes/`, verifies the official SHA-256, and retries once. Java 25 class-file incompatibility selects Java 21; a Paper maximum of Java 15 selects Java 11. For the specific Forge `ManifestEntryVerifier` incompatibility, method-signature failures on Java 8, 11, and 17 select known earlier patch releases of the same Java feature (8u312, 11.0.13, and 17.0.1). Module-access failures, and this signature error on other feature versions, fall back to Java 8u312 for the affected legacy Forge. These are signature-specific compatibility rules, not a guarantee that every Java patch regression can be detected or fixed automatically. Other crashes do not trigger downloads or retries. Installer and proxy startup failures are not automatically retried.

For Fabric, Forge, and NeoForge backends behind Velocity, set `lobby.minecraft-version` or `plugin.<name>.minecraft-version` to the exact Minecraft version if it cannot be detected from the JAR filename/metadata. On first startup the launcher downloads the exact compatible forwarding mod from Modrinth into that backend's `mods/` directory, verifies its SHA-512 hash, and writes the shared Velocity secret to the mod's own config. Fabric also receives Fabric API. NeoForge prefers NeoVelocity and automatically falls back to Proxy Compatible Forge when NeoVelocity has no release for the exact version. Fabric uses `config/FabricProxy-Lite.toml`, NeoVelocity uses `config/neovelocity-common.toml`, and Proxy Compatible Forge uses `config/proxy-compatible-forge.toml` (not a generic `config.yml`). Only versions published for that exact loader and Minecraft version are supported; missing releases stop startup with an error. These automatic modded forwarding setups require Velocity modern mode. Arclight/Mohist may be run with modern, legacy, or none; the launcher configures Arclight's `arclight.conf` and Mohist's `mohist-config/mohist.yml` for the selected mode. Automatic forwarding for Fabric/Forge/NeoForge is not available with BungeeCord or Waterfall. Old Forge versions that require a modified Velocity proxy are rejected.
For an already installed Forge/NeoForge backend, the launcher derives Minecraft's exact version from its installed server library and rejects a conflicting `minecraft-version` setting. This prevents installing a forwarding mod for (for example) 1.21.1 on an installed 1.21 server.
NeoVelocity's embedded NeoForge dependency range is checked as well as Modrinth's version tags; if the tags claim compatibility but the mod's metadata excludes the installed Minecraft version, the launcher tries Proxy Compatible Forge instead.

Forge/NeoForge installer runs are checked for the libraries referenced by their launch profile. If an installer was configured and a previously generated profile has missing libraries, the launcher reruns the installer to repair it. If needed, provide an exact-version `libraries.zip` with `lobby.libs` or `plugin.<name>.libs`; its entries may be under a top-level `libraries/` directory or directly contain the library paths. The archive is extracted only into that backend's `run/<name>/libraries/`. Remaining missing files are reported before the server starts.
JAR entries in the archive and referenced JAR dependencies are checked for valid ZIP/JAR structure, so a truncated or corrupt server JAR is reported before Java tries to launch it.
NeoForge launch-profile validation ignores Java module-access options such as `--add-opens` and checks comma-separated legacy classpath entries against the installed libraries, avoiding false missing-library errors.
Legacy Forge releases such as 1.16.4 must be installed with their matching server installer; the launcher uses the generated Forge JAR in the server run directory and will not mistake a similarly named file under `libraries/` for the server executable. Modern Forge uses its generated argument profile (`unix_args.txt` on Linux/macOS, `win_args.txt` on Windows); Windows batch tokens such as `%*` and `pause` are not Java launch arguments.
With `launch-args=auto`, relative launch paths are resolved from that backend's `run/<name>/` working directory. The launcher prints the selected arguments and working directory before starting it. Forge may still load dependencies from the `libraries/` directory as required by its profile.

Minecraft versions from 1.8 through 26.3 can be selected by supplying a matching server build; the launcher does not enforce that range or guarantee compatibility with any particular Java runtime. A Limbo type is treated as a standalone TCP backend; its `server.properties` receives the launcher-assigned bind address/port and BungeeCord or Velocity forwarding settings. Proxy plugins named Limbo belong in `plugins/proxy/`.

For installation, Java/version compatibility, RAM planning, multi-server setup, commands, networking, and troubleshooting, see [KULLANIM_KILAVUZU.txt](./KULLANIM_KILAVUZU.txt).

English version: [USER_GUIDE_EN.txt](./USER_GUIDE_EN.txt).

Turkish project overview: [TANITIM.txt](./TANITIM.txt).
