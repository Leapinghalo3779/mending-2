package com.example.maceupgrade;

import org.bukkit.Location;
import org.bukkit.Material;
import org.bukkit.NamespacedKey;
import org.bukkit.Particle;
import org.bukkit.Sound;
import org.bukkit.World;
import org.bukkit.enchantments.Enchantment;
import org.bukkit.entity.LivingEntity;
import org.bukkit.entity.Player;
import org.bukkit.event.EventHandler;
import org.bukkit.event.Listener;
import org.bukkit.event.enchantment.EnchantItemEvent;
import org.bukkit.event.enchantment.PrepareItemEnchantEvent;
import org.bukkit.event.entity.EntityDamageByEntityEvent;
import org.bukkit.event.entity.PlayerDeathEvent;
import org.bukkit.event.inventory.PrepareAnvilEvent;
import org.bukkit.inventory.ItemStack;
import org.bukkit.inventory.meta.ItemMeta;
import org.bukkit.persistence.PersistentDataContainer;
import org.bukkit.persistence.PersistentDataType;
import org.bukkit.plugin.java.JavaPlugin;
import org.bukkit.util.Vector;

import java.util.HashMap;
import java.util.Map;
import java.util.UUID;

public class MaceUpgrade extends JavaPlugin implements Listener {

    // Index = number of kills (0 to 5). Kills above 5 stay at the level for 5.
    //                                    kills: 0  1  2  3  4  5
    private static final int[] DENSITY_LEVEL = {0, 1, 2, 3, 4, 5};
    private static final int[] WIND_BURST_LEVEL = {0, 0, 0, 1, 2, 2};

    // Shockwave settings (change these to tune it)
    private static final int SHOCKWAVE_KILLS = 3;        // kills needed to unlock
    // Gets stronger with every level.        kills: 0    1    2    3    4    5
    private static final double[] SHOCKWAVE_RADIUS   = {0,   0,   0,   3.5, 4.5, 6.0}; // blocks
    private static final double[] SHOCKWAVE_STRENGTH = {0,   0,   0,   1.1, 1.6, 2.4}; // sideways push
    private static final double[] SHOCKWAVE_LIFT     = {0,   0,   0,   0.4, 0.5, 0.8}; // upward push
    private static final long SHOCKWAVE_COOLDOWN_MS = 1500;

    private NamespacedKey killsKey;
    private final Map<UUID, Long> lastShockwave = new HashMap<>();

    @Override
    public void onEnable() {
        killsKey = new NamespacedKey(this, "mace_kills");
        getServer().getPluginManager().registerEvents(this, this);
    }

    @EventHandler
    public void onPlayerDeath(PlayerDeathEvent event) {
        Player killer = event.getEntity().getKiller();
        if (killer == null) return;

        ItemStack mace = killer.getInventory().getItemInMainHand();
        if (mace.getType() != Material.MACE) return;

        ItemMeta meta = mace.getItemMeta();
        if (meta == null) return;

        // Kill count is saved on the mace itself
        PersistentDataContainer data = meta.getPersistentDataContainer();
        int kills = data.getOrDefault(killsKey, PersistentDataType.INTEGER, 0) + 1;
        data.set(killsKey, PersistentDataType.INTEGER, kills);

        int index = Math.min(kills, DENSITY_LEVEL.length - 1);
        applyLevel(meta, Enchantment.DENSITY, DENSITY_LEVEL[index]);
        applyLevel(meta, Enchantment.WIND_BURST, WIND_BURST_LEVEL[index]);
        applyLevel(meta, Enchantment.MENDING, 1);
        applyLevel(meta, Enchantment.UNBREAKING, 3);

        mace.setItemMeta(meta);

        killer.sendMessage("Your mace now has " + kills + " kill" + (kills == 1 ? "" : "s")
                + " (Density " + DENSITY_LEVEL[index]
                + (WIND_BURST_LEVEL[index] > 0 ? ", Wind Burst " + WIND_BURST_LEVEL[index] : "")
                + ")");

        if (kills == SHOCKWAVE_KILLS) {
            killer.sendMessage("Shockwave unlocked! Your hits now knock nearby players back.");
        }
    }

    @EventHandler
    public void onHit(EntityDamageByEntityEvent event) {
        if (!(event.getDamager() instanceof Player attacker)) return;

        ItemStack mace = attacker.getInventory().getItemInMainHand();
        if (mace.getType() != Material.MACE) return;

        ItemMeta meta = mace.getItemMeta();
        if (meta == null) return;

        int kills = meta.getPersistentDataContainer()
                .getOrDefault(killsKey, PersistentDataType.INTEGER, 0);
        if (kills < SHOCKWAVE_KILLS) return;

        int level = Math.min(kills, SHOCKWAVE_RADIUS.length - 1);

        // Cooldown so it can't be spammed
        long now = System.currentTimeMillis();
        Long last = lastShockwave.get(attacker.getUniqueId());
        if (last != null && now - last < SHOCKWAVE_COOLDOWN_MS) return;
        lastShockwave.put(attacker.getUniqueId(), now);

        Location center = event.getEntity().getLocation();
        World world = center.getWorld();

        world.spawnParticle(Particle.EXPLOSION, center.clone().add(0, 1, 0), 1);
        world.playSound(center, Sound.ENTITY_GENERIC_EXPLODE, 1.0f, 1.2f);

        for (LivingEntity target : world.getNearbyLivingEntities(center, SHOCKWAVE_RADIUS[level])) {
            // Skip the person swinging and the one who was hit (they already get normal knockback)
            if (target.equals(attacker) || target.equals(event.getEntity())) continue;

            Vector push = target.getLocation().toVector().subtract(center.toVector());
            push.setY(0);
            if (push.lengthSquared() < 0.0001) continue;

            push.normalize().multiply(SHOCKWAVE_STRENGTH[level]).setY(SHOCKWAVE_LIFT[level]);
            target.setVelocity(push);
        }
    }

    // ---- Block other ways of enchanting the mace ----

    // Anvil: no combining two maces, no enchanted books on a mace.
    // Renaming and repairing (e.g. with breeze rods) still work.
    @EventHandler
    public void onPrepareAnvil(PrepareAnvilEvent event) {
        ItemStack first = event.getInventory().getFirstItem();
        ItemStack second = event.getInventory().getSecondItem();
        if (first == null || second == null) return;
        if (first.getType() != Material.MACE) return;

        if (second.getType() == Material.MACE || second.getType() == Material.ENCHANTED_BOOK) {
            event.setResult(null);
        }
    }

    // Enchanting table: no enchant options for maces
    @EventHandler
    public void onPrepareEnchant(PrepareItemEnchantEvent event) {
        if (event.getItem().getType() == Material.MACE) {
            event.setCancelled(true);
        }
    }

    @EventHandler
    public void onEnchant(EnchantItemEvent event) {
        if (event.getItem().getType() == Material.MACE) {
            event.setCancelled(true);
        }
    }

    private void applyLevel(ItemMeta meta, Enchantment enchantment, int level) {
        if (level > 0) {
            meta.addEnchant(enchantment, level, true);
        }
    }
}
