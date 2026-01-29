const {
  Client,
  GatewayIntentBits,
  Partials,
  EmbedBuilder,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  PermissionsBitField,
  ChannelType
} = require("discord.js");

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
    GatewayIntentBits.GuildMembers
  ],
  partials: [Partials.Channel]
});

const prefix = "!";
const TOKEN = "SEU_TOKEN_AQUI";

/* ================= CONFIG EM MEMÓRIA ================= */
let staffRoleId = null;
let ticketCategoryId = null;
let categorias = [];

/* ================= READY ================= */
client.once("ready", () => {
  console.log(`✅ Bot conectado como ${client.user.tag}`);
});

/* ================= COMANDOS ================= */
client.on("messageCreate", async (message) => {
  if (!message.guild || message.author.bot) return;
  if (!message.content.startsWith(prefix)) return;

  const args = message.content.slice(prefix.length).trim().split(/ +/);
  const cmd = args.shift()?.toLowerCase();

  if (cmd === "ajuda" || cmd === "ajudar") {
    const embed = new EmbedBuilder()
      .setTitle("📌 Comandos do Bot")
      .setColor(0x2f3136)
      .setDescription(
        "`!ajuda` → Mostrar comandos\n" +
        "`!setstaff @cargo` → Definir staff\n" +
        "`!setcategoria Nome` → Categoria dos tickets\n" +
        "`!addcategoria Nome Emoji` → Categoria do painel\n" +
        "`!painel` → Enviar painel"
      );

    return message.reply({ embeds: [embed] });
  }

  if (cmd === "setstaff") {
    if (!message.member.permissions.has(PermissionsBitField.Flags.Administrator))
      return message.reply("❌ Apenas administradores.");

    const role = message.mentions.roles.first();
    if (!role) return message.reply("❌ Mencione um cargo.");

    staffRoleId = role.id;
    return message.reply("✅ Cargo staff definido.");
  }

  if (cmd === "setcategoria") {
    const nome = args.join(" ");
    if (!nome) return message.reply("❌ Informe o nome.");

    let cat = message.guild.channels.cache.find(
      c => c.type === ChannelType.GuildCategory && c.name === nome
    );

    if (!cat) {
      cat = await message.guild.channels.create({
        name: nome,
        type: ChannelType.GuildCategory
      });
    }

    ticketCategoryId = cat.id;
    return message.reply("✅ Categoria definida.");
  }

  if (cmd === "addcategoria") {
    if (!args[0] || !args[1])
      return message.reply("❌ Use: !addcategoria <nome> <emoji>");

    categorias.push({ nome: args[0], emoji: args[1] });
    return message.reply("✅ Categoria adicionada.");
  }

  if (cmd === "painel") {
    if (categorias.length === 0)
      return message.reply("❌ Nenhuma categoria configurada.");

    const embed = new EmbedBuilder()
      .setTitle("🎫 Painel de Tickets")
      .setDescription("Clique em um botão para abrir um ticket")
      .setColor(0x2f3136);

    const row = new ActionRowBuilder();

    categorias.slice(0, 5).forEach(cat => {
      row.addComponents(
        new ButtonBuilder()
          .setCustomId(`ticket_${cat.nome}`)
          .setLabel(cat.nome)
          .setEmoji(cat.emoji)
          .setStyle(ButtonStyle.Primary)
      );
    });

    return message.channel.send({ embeds: [embed], components: [row] });
  }
});

/* ================= INTERAÇÕES ================= */
client.on("interactionCreate", async (interaction) => {
  if (!interaction.isButton()) return;

  /* ===== ABRIR TICKET ===== */
  if (interaction.customId.startsWith("ticket_")) {
    if (!staffRoleId || !ticketCategoryId) {
      return interaction.reply({ content: "❌ Sistema não configurado.", ephemeral: true });
    }

    const motivo = interaction.customId.replace("ticket_", "");
    const staffRole = interaction.guild.roles.cache.get(staffRoleId);
    const category = interaction.guild.channels.cache.get(ticketCategoryId);

    const canal = await interaction.guild.channels.create({
      name: `ticket-${interaction.user.username}`,
      type: ChannelType.GuildText,
      parent: category,
      permissionOverwrites: [
        { id: interaction.guild.id, deny: [PermissionsBitField.Flags.ViewChannel] },
        {
          id: interaction.user.id,
          allow: [
            PermissionsBitField.Flags.ViewChannel,
            PermissionsBitField.Flags.SendMessages
          ]
        },
        { id: staffRole.id, allow: [PermissionsBitField.Flags.ViewChannel] }
      ]
    });

    const embed = new EmbedBuilder()
      .setTitle("🎟 Ticket Aberto")
      .setColor(0x5865f2)
      .setDescription(
        `👤 **Usuário:** ${interaction.user}\n` +
        `📌 **Motivo:** \`${motivo}\`\n` +
        `🧑‍💼 **Staff:** _Ninguém assumiu_\n\n` +
        `📅 **Aberto em:** <t:${Math.floor(Date.now() / 1000)}:F>`
      );

    const row = new ActionRowBuilder().addComponents(
      new ButtonBuilder().setCustomId("assumir").setLabel("👤 Assumir").setStyle(ButtonStyle.Success),
      new ButtonBuilder().setCustomId("cliente").setLabel("📣 Chamar Cliente").setStyle(ButtonStyle.Primary),
      new ButtonBuilder().setCustomId("staff").setLabel("🔔 Chamar Staff").setStyle(ButtonStyle.Secondary),
      new ButtonBuilder().setCustomId("fechar").setLabel("🔒 Fechar").setStyle(ButtonStyle.Danger)
    );

    await canal.send({
      content: `${staffRole} | ${interaction.user}`,
      embeds: [embed],
      components: [row]
    });

    return interaction.reply({ content: "✅ Ticket criado!", ephemeral: true });
  }

  /* ===== ASSUMIR ===== */
  if (interaction.customId === "assumir") {
    if (!interaction.member.roles.cache.has(staffRoleId)) {
      return interaction.reply({ content: "❌ Apenas staff pode assumir.", ephemeral: true });
    }

    const embed = EmbedBuilder.from(interaction.message.embeds[0]);
    embed.setDescription(
      embed.data.description.replace(
        /🧑‍💼 \*\*Staff:\*\* .*/,
        `🧑‍💼 **Staff:** ${interaction.user}`
      )
    );

    const row = new ActionRowBuilder().addComponents(
      new ButtonBuilder().setCustomId("assumido").setLabel("✅ Assumido").setStyle(ButtonStyle.Secondary).setDisabled(true),
      new ButtonBuilder().setCustomId("cliente").setLabel("📣 Chamar Cliente").setStyle(ButtonStyle.Primary),
      new ButtonBuilder().setCustomId("staff").setLabel("🔔 Chamar Staff").setStyle(ButtonStyle.Secondary),
      new ButtonBuilder().setCustomId("fechar").setLabel("🔒 Fechar").setStyle(ButtonStyle.Danger)
    );

    await interaction.message.edit({ embeds: [embed], components: [row] });
    return interaction.reply({ content: "✅ Ticket assumido.", ephemeral: true });
  }

  /* ===== CHAMAR CLIENTE ===== */
  if (interaction.customId === "cliente") {
    return interaction.reply({
      content: `📣 ${interaction.channel.permissionOverwrites.cache
        .filter(p => p.type === 1)
        .map(p => `<@${p.id}>`)
        .join(" ")} você foi chamado no ticket!`
    });
  }

  /* ===== CHAMAR STAFF ===== */
  if (interaction.customId === "staff") {
    return interaction.reply({
      content: `🔔 <@&${staffRoleId}> staff chamada para este ticket!`
    });
  }

  /* ===== FECHAR ===== */
  if (interaction.customId === "fechar") {
    await interaction.reply({ content: "🔒 Ticket será fechado em 5 segundos...", ephemeral: true });
    setTimeout(() => interaction.channel.delete().catch(() => {}), 5000);
  }
});

/* ================= LOGIN ================= */
client.login(TOKEN);
