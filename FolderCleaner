import os
import datetime
from datetime import datetime
import os.path
import shutil
import time

import smtplib
from email.message import EmailMessage


from settings import *

current_day = datetime.now().strftime("%Y%m%d")  # текущий день на начало старта скрипта


def log(s, need_to_write: bool = True):
    """ Вывод сообщения в командную строку и, если нужно, то и запись в файл """
    now = datetime.now()
    msg = f'{now.strftime("%Y-%m-%d %H:%M:%S")}: {s}'
    print(msg)
    if need_to_write:
        fn = f'{now.strftime("%Y%m%d")}_log.txt'
        log_dir = 'logs'
        if not os.path.exists(log_dir):
            os.mkdir(log_dir)
        fn = os.path.join(log_dir, fn)
        with open(fn, 'a', encoding='utf-8') as f:
            f.write(f'{msg}\n')


def get_files(path, is_scan_subfolder=False):
    """ Получить список файлов в заданной папке. Если вторым параметром передать True, то будет сканировать подпапки """
    found_files = []
    if is_scan_subfolder:
        # сканировать и подпапки
        try:
            for root, dirs, files in os.walk(path):
                for file in files:
                    found_files.append(os.path.join(root, file))
        except Exception as e:
            log(f'Ошибка получения списка файлов: {e}')
    else:
        try:
            with os.scandir(path) as files:
                found_files = [os.path.join(path, file.name) for file in files if file.is_file()]
        except Exception as e:
            log(f'Ошибка получения списка файлов: {e}')

    return found_files


def move_files(found_files, dest_folder):
    if not os.path.isdir(dest_folder):
        log(f'Не найдена папка для перемещения: {dest_folder}')
        return False

    is_error = False
    for num, file_name in enumerate(found_files, start=1):
        try:
            log(f'[{num}/{len(found_files)}] Работа с файлом: "{file_name}"')
            shutil.move(file_name, dest_folder)
            log(f'Успешно перемещен "{file_name}" в папку "{dest_folder}".')
        except Exception as e:
            log(f'Ошибка перемещения файла "{file_name}": {e}')
            is_error = True

    return not is_error


def send_log_to_email(fn):
    log('Отправка письма с уведомлением.')
    msg = EmailMessage()

    msg['From'] = FROM_EMAIL
    msg['To'] = TO_EMAIL
    msg['Subject'] = 'Уведомление от скрипта "FolderCleaner"'

    body = 'Скрипт работает...'

    try:
        msg.set_content(body)
        msg.add_attachment(open(fn, "r", encoding='utf-8').read(), filename=os.path.basename(fn))

        server = smtplib.SMTP_SSL('smtp.mail.ru', 465)
        server.login(FROM_EMAIL, FROM_PASSWORD)
        server.send_message(msg)
        server.quit()
        return True
    except Exception as e:
        log(f'Ошибка отправки письма: {str(e)}')
        return False


def send_notify_to_email(old_files):
    log('Отправка письма с уведомлением об ошибках удаления')
    if not old_files:
        log('Нечего отправлять - список старых файлов пуст!')
        return False

    msg = EmailMessage()

    msg['From'] = FROM_EMAIL
    msg['To'] = TO_EMAIL
    msg['Subject'] = 'Уведомление от скрипта "FolderCleaner"'

    msg_files = "\n".join(old_files)
    body = f'Ошибки при удалении файлов: {msg_files}'

    try:
        msg.set_content(body)
        server = smtplib.SMTP_SSL('smtp.mail.ru', 465)
        server.login(FROM_EMAIL, FROM_PASSWORD)
        server.send_message(msg)
        server.quit()
        return True
    except Exception as e:
        log(f'Ошибка отправки письма: {str(e)}')
        return False


def get_old_files(found_files):
    if not found_files:
        return None

    old_files = []
    for fn in found_files:
        date_created = datetime.utcfromtimestamp(os.path.getmtime(fn))
        delta_days = (datetime.now() - date_created).days
        if delta_days >= MAX_DELTA_DAYS:
            old_files.append(fn)

    return old_files


def main():
    is_moved = True
    global current_day
    errors_count = 0
    if_first_check = True

    while True:
        if not if_first_check:
            log('Пауза перед следующим циклом проверки папки...')
            time.sleep(PAUSE_REFRESH_FOLDER)

            now_day = datetime.now().strftime("%Y%m%d")
            if now_day != current_day:
                log('Сработал переход на новый день и будет отправлено уведомление на почту!')
                fn = os.path.join('logs', f'{current_day}_log.txt')
                send_log_to_email(fn)
                current_day = now_day

        if_first_check = False
        if is_moved:
            log('Ожидание новых файлов...')

        while True:  # цикл на попытки перемещения файлов
            found_files = get_files(OBSERVATION_FOLDER, SCAN_SUBFOLDERS)
            if not found_files:
                is_moved = False
                break

            old_files = get_old_files(found_files)
            if not old_files:
                log('Старых файлов не обнаружено!')
                is_moved = False
                break

            log(f'Количество файлов старше {MAX_DELTA_DAYS} дней: {len(old_files)}')
            if move_files(old_files, MOVE_FOLDER):
                is_moved = True
                errors_count = 0
                break

            errors_count += 1
            log(f'Не удалось переместить файлы. Количество подряд идущих ошибок = {errors_count}.')
            if errors_count >= MAX_ERRORS_COUNT:
                log('Сработал лимит на количество подряд идуших ошибок. Отправляем уведомление на почту')
                send_notify_to_email(old_files)
                errors_count = 0
                break

            log(f'Делаем паузу в 1 минуту...')
            time.sleep(60)
            is_moved = False


if __name__ == '__main__':
    main()
