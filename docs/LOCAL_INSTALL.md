### Установка Kubernetes локально

1. Ставим себе Docker Desktop https://www.docker.com/products/docker-desktop/
2. Ставим себе WSL (аналог виртуальной машины Linux на Windows) https://aka.ms/wslinstall
3. Ставим себе Ubuntu в WSL из Microsoft Store https://apps.microsoft.com/store/search?publisher=Canonical%20Group%20Limited
4. Перезапускаем компьютер
5. Запускаем Ubuntu и ждем завершение установки
6. Запускаем Docker Desktop
7. Подключаем Ubuntu к Docker Desktop. Для этого в его настройках переходим в раздел `Resorces` и там в `WSL Integration` включаем интеграцию с Ubuntu
8. Включаем Kubernetes в настройках Docker Desktop (если хотим смотреть за прогрессом загрузки выходим из настроек и смотрим как появляются новые Images)
9. Когда у нас иконка Kubernetes (штурвал от корабля) внизу станет зеленой это значит что мы можем зайти в Ubuntu и выполнить команду `kubectl version --short` которая должна вывести версию Kubernetes
