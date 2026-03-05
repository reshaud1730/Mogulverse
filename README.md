# Mogulverse
The Entrepreneur Simulation Discord Game
@bot.command(name='leaderboard')
async def show_leaderboard(ctx):
    """Displays the top moguls based on current portfolio value."""
    c = conn.cursor()
    c.execute("SELECT user_id, industry, balance FROM players ORDER BY balance DESC LIMIT 5")
    top_players = c.fetchall()

    embed = discord.Embed(title="🏆 Mogulverse Weekly Top 5", color=discord.Color.gold())
    
    for i, (uid, industry, bal) in enumerate(top_players, 1):
        user = await bot.fetch_user(uid)
        embed.add_field(
            name=f"{i}. {user.name} ({industry})", 
            value=f"Portfolio: {bal:,.2f} Credits", 
            inline=False
        )
    
    await ctx.send(embed=embed)

@bot.command(name='payout_and_reset')
@commands.has_permissions(administrator=True) # Only you can trigger the weekly reset
async def weekly_reset(ctx):
    """Distributes Trickle Fund rewards and resets the board."""
    c = conn.cursor()
    
    # 1. Identify the Winner
    c.execute("SELECT user_id, balance FROM players ORDER BY balance DESC LIMIT 1")
    winner = c.fetchone()
    
    if not winner:
        return await ctx.send("No active players to reward.")

    # 2. Calculate Payout (e.g., 50% of the current Trickle Fund)
    c.execute("SELECT value FROM global_stats WHERE stat_name = 'trickle_fund'")
    fund_total = c.fetchone()[0]
    prize_pool = fund_total * 0.50
    
    # 3. Reward the Winner & Reset Global Stats
    # Add prize to winner's balance (they start the next week with a bonus)
    new_start_bal = 5000.0 + prize_pool
    
    # 4. Reset everyone else to the baseline $5,000
    c.execute("UPDATE players SET balance = 5000.0")
    c.execute("UPDATE players SET balance = ? WHERE user_id = ?", (new_start_bal, winner[0]))
    
    # Deduct the prize from the Trickle Fund
    c.execute("UPDATE global_stats SET value = value - ? WHERE stat_name = 'trickle_fund'", (prize_pool,))
    
    conn.commit()

    winner_user = await bot.fetch_user(winner[0])
    await ctx.send(f"🎊 **THE FISCAL WEEK HAS ENDED!** 🎊\n\n"
                   f"Congratulations to **{winner_user.name}** for topping the charts!\n"
                   f"They have been awarded **{prize_pool:,.2f} Credits** from the Trickle Fund.\n"
                   f"All portfolios have been re-indexed for the new week. Good luck, Moguls!")
