#!/usr/bin/env node
/**
 * VKMBOT v14.0 — NODE.JS FINAL
 * Запускать на: Render, Heroku, Railway, VPS.
 * Не запускать на Cloudflare!
 */

const TelegramBot = require('node-telegram-bot-api');
const ytdl = require('yt-dlp-exec');
const fs = require('fs');
const os = require('os');
const path = require('path');
const { v4: uuidv4 } = require('uuid');

// === КОНФИГУРАЦИЯ ===
// Вставьте токен через переменные окружения или сюда
const API_TOKEN = process.env.BOT_TOKEN || "6409245799:AAEAj-E0BR0Kq4MhxTIZEpUNC6N7yHW8MRg";
const ADMIN_ID = 1979411532;

// Инициализация бота (Polling - сам опрашивает сервер, не нужен Webhook)
const bot = new TelegramBot(API_TOKEN, { polling: true });

// === СОСТОЯНИЕ ===
const DB = {
  users: new Set(),
  downloads: 0,
  favorites: new Map()
};

// === КЛАВИАТУРЫ ===
function getMainKeyboard(userId) {
  const menu = [
    [{text: "🔍 Поиск музыки"}, {text: "🔥 Топ"}],
    [{text: "❤️ Избранное"}, {text: "📊 Профиль"}],
    [{text: "⚙️ Настройки"}]
  ];
  if (userId === ADMIN_ID) menu.push([{text: "👑 Админ Панель"}]);
  return { keyboard: menu, resize_keyboard: true };
}

function getAdminKeyboard() {
  return {
    keyboard: [
      [{text: "📊 Статистика"}, {text: "👥 Юзеры"}],
      [{text: "📢 Рассылка"}, {text: "🔙 Назад"}]
    ],
    resize_keyboard: true
  };
}

// === ЛОГИКА МУЗЫКИ (yt-dlp) ===
// Это работает только на серверах (VPS, Render), где установлен yt-dlp
async function downloadVideo(videoId) {
  const tempDir = os.tmpdir();
  const fileName = `${uuidv4()}.mp3`;
  const outputPath = path.join(tempDir, fileName);
  
  try {
    // Запускаем yt-dlp
    await ytdl(`https://www.youtube.com/watch?v=${videoId}`, {
      output: outputPath,
      format: 'bestaudio[ext=m4a]/bestaudio/best',
      extractAudio: true,
      audioFormat: 'mp3',
      quiet: true,
      noWarnings: true
    });

    if (fs.existsSync(outputPath)) {
      return outputPath;
    }
    return null;
  } catch (e) {
    console.error("Download Error:", e);
    return null;
  }
}

// === ОБРАБОТЧИКИ БОТА ===

bot.onText(/\/start/, async (msg) => {
  const chatId = msg.chat.id;
  DB.users.add(chatId);
  await bot.sendMessage(chatId, "🎵 *Бот готов!*\n\nДля скачивания просто напишите название трека.", {
    parse_mode: "Markdown",
    reply_markup: getMainKeyboard(chatId)
  });
});

bot.on('message', async (msg) => {
  const chatId = msg.chat.id;
  const text = msg.text;
  const isAdm = chatId === ADMIN_ID;
  
  DB.users.add(chatId);

  // === НАВИГАЦИЯ ===
  if (text === "🔍 Поиск музыки") {
    bot.sendMessage(chatId, "🔍 *Введите трек*", getMainKeyboard(chatId));
  }
  else if (text === "🔥 Топ") {
    bot.sendMessage(chatId, "🔥 *Популярное*", getMainKeyboard(chatId));
  }
  else if (text === "❤️ Избранное") {
    const favs = DB.favorites.get(chatId) || [];
    if (!favs.length) { bot.sendMessage(chatId, "📂 Пусто", getMainKeyboard(chatId)); return; }
    
    // Формируем кнопки из избранных
    const btns = favs.map((f, i) => [{text: `🎵 ${f.title}`, callback_data: `dl_fav_${i}`}]);
    bot.sendMessage(chatId, "❤️ *Избранное*", {
      reply_markup: { inline_keyboard: btns },
      parse_mode: "Markdown"
    });
  }
  else if (text === "📊 Профиль") {
    bot.sendMessage(chatId, `👤 Профиль\n📥 ${DB.downloads}\n🆔 ${chatId}`, getMainKeyboard(chatId));
  }
  else if (text === "⚙️ Настройки") {
    bot.sendMessage(chatId, "⚙️ *Настройки*", getMainKeyboard(chatId));
  }
  else if (text === "👑 Админ Панель" && isAdm) {
    bot.sendMessage(chatId, "👑 *Админка*", getAdminKeyboard());
  }
  else if (text === "🔙 Назад") {
    bot.sendMessage(chatId, "🎵 *Меню*", getMainKeyboard(chatId));
  }

  // === АДМИН ФУНКЦИИ ===
  else if (isAdm && text.startsWith("/broadcast ")) {
    const msg = text.replace("/broadcast ", "");
    let s = 0;
    for(const u of DB.users) { 
      try{ bot.sendMessage(u, `📢 *Рассылка:*\n${msg}`, {parse_mode: "Markdown"}); s++; }catch(e){} 
    }
    bot.sendMessage(chatId, `✅ Отправлено: ${s}`, getAdminKeyboard());
  }
  else if (isAdm && text === "📊 Статистика") {
    bot.sendMessage(chatId, `👥 ${DB.users.size}\n⬇️ ${DB.downloads}`, getAdminKeyboard());
  }

  // === ПОИСК (ТЕКСТ) ===
  else if (text && text.length > 2 && !text.startsWith("/")) {
    // В Node.js без внешнего API поиска (чтобы не зависеть от инстансов),
    // мы можем только скачивать по прямой ссылке или ID.
    // Если нужно искать по названию - нужен Python+yt-dlp или внешний API.
    
    // Для стабильности: запросим у пользователя ID видео, если текст похож на ID, иначе ошибку.
    // Либо можно использовать бесплатный API для поиска Invidious.
    
    bot.sendMessage(chatId, "⏳ *Ищу...* (через Invidious)", {parse_mode: "Markdown"});
    
    try {
      const searchUrl = `https://invidious.snopyta.org/api/v1/search?q=${encodeURIComponent(text)}`;
      const response = await fetch(searchUrl);
      const data = await response.json();
      
      if (data && data.length > 0) {
        const btns = data.slice(0, 5).map(r => ([
          {text: `🎵 ${r.title.substring(0,30)}...`, callback_data: `dl_yt_${r.videoId}`},
          {text: "❤️", callback_data: `fav_yt_${r.videoId}_${r.title.substring(0,20)}`}
        ])).flat();
        
        bot.sendMessage(chatId, `🎧 ${text}`, { reply_markup: { inline_keyboard: [btns] } });
      } else {
        bot.sendMessage(chatId, "❌ Не найдено", getMainKeyboard(chatId));
      }
    } catch(e) {
      bot.sendMessage(chatId, "❌ Ошибка поиска (Invidius недоступен)", getMainKeyboard(chatId));
    }
  }
});

bot.on('callback_query', async (query) => {
  const chatId = query.message.chat.id;
  const msgId = query.message.message_id;
  const data = query.data;

  bot.answerCallbackQuery(query.id, { text: "⏳ Качаю..." });

  if (data.startsWith("dl_yt_")) {
    const id = data.replace("dl_yt_", "");
    
    try {
      bot.editMessageText(chatId, msgId, "⏳ *Скачиваю...* (может занять минуту)", {parse_mode: "Markdown"});
      
      // Скачивание через yt-dlp
      const filePath = await downloadVideo(id);
      
      if (filePath) {
        await bot.sendAudio(chatId, filePath, { caption: "🎵 VKMBOT" });
        DB.downloads++;
        
        // Удаляем файл
        fs.unlink(filePath, (err) => {});
        
        bot.sendMessage(chatId, "✅ Готово!", getMainKeyboard(chatId));
      } else {
        bot.sendMessage(chatId, "❌ Ошибка скачивания (yt-dlp упал)", getMainKeyboard(chatId));
      }
    } catch (e) {
      bot.sendMessage(chatId, "❌ Ошибка", getMainKeyboard(chatId));
    }
  }
  else if (data.startsWith("dl_fav_")) {
    const idx = parseInt(data.split("_")[2]);
    const list = DB.favorites.get(chatId);
    if (list && list[idx]) {
      // Повторяем логику скачивания для избранного
      bot.answerCallbackQuery(query.id, { text: "⏳" });
      const filePath = await downloadVideo(list[idx].id);
      if (filePath) {
        await bot.sendAudio(chatId, filePath, { caption: "🎵" });
        bot.sendMessage(chatId, "✅ Готово!", getMainKeyboard(chatId));
        fs.unlink(filePath, () => {});
      }
    }
  }
  else if (data.startsWith("fav_yt_")) {
    const parts = data.replace("fav_yt_", "").split("_");
    const id = parts[0];
    const title = parts.slice(1).join("_");
    const list = DB.favorites.get(chatId) || [];
    list.push({ title, id });
    DB.favorites.set(chatId, list);
    bot.answerCallbackQuery(query.id, { text: "✅ Сохранено", show_alert: true });
  }
});

console.log("VKMBOT Node.js version started...");
