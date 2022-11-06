# kws_project_2

Выполнил Кондратьев Захар, 192

Итоговая модель со сжатием в 10.113209346685741 раз и ускорением в 9.651004992958647, также реализован класс для стриминга.

## Воспроизведение результатов
Код и логи запусков можно найти в блокноте dla_homework2.ipynb.
1) Последовательный запуск ячеек до раздела Training teacher
2) Если нужно обучить с нуля базовую модель, последовательный запуск ячеек раздела Training teacher, иначе можно их пропустить и далее будет загрузка весов обученной модели (model_teacher)
3) Последовательный запуск ячеек раздела Distillation для воспроизведение экспериментов
4) Последовательный запуск ячеек раздела Streaming для проверки стриминговой модели на основе дистиллированной CRNN и её сохранения в jit формате

## Чекпоинты

- model_teacher, базовая модель, обученная до качества менее 5e-5
- model_best, дистиллированная модель (сжатие в ~6.88 раз и ускорение в ~9.65), обученная с помощью Dark Knowledge Distillation с model_teacher
- model_scripted.pt, стриминговая модель, обёртка над model_best с квантизацией, сохранённая в jit формате

## Описание экспериментов
Полных список гиперпараметров запусков можно найти в блокноте.
### Dark Knowledge Distillation
Модель обучаются с двумя функциями потерь: cross_entropy, где logits берутся от обучаемой модели, а labels из предсказаний модели учителя (loss1) или из данных (loss2)  
Схематически:

    batch_X -> model_teacher -> softmax(T=t) \
                                              -> loss1
    batch_X ->     model    --> softmax(T=t) /
                            \
                             -> softmax(T=1) \
                                              -> loss2
                                  batch_y -> /  

В данной постановке получены веса для финальной модели (сжатие в ~6.88 раз и ускорение в ~9.65). Ключевые изменения:  
- kernel_size[0]=7, stride[0]=6, cnn_out_channels=3 вместо kernel_size[0]=5, stride[0]=2, cnn_out_channels=8, что позволило значительно уменьшить размерность входных векторов перед GRU, так эта размерность равна  
((40 - kernel_size[0]) // stride[0] + 1) * cnn_out_channels
- hidden_size=12 вместо hidden_size=32, что позволило значительно уменьшить также выходную размерность векторов GRU
- stride[1]=10 вместо stride[1]=8, не влияет на размер модели, но значительно ускоряет

### Attention Distillation + Dark Knowledge Distillation
Аналогично предыдущему эксперименту, но теперь будет три функции потерь, добавляется ещё cross_entropy между предсказанными коэффициентами attention моделей учителя и ученика.  
Разумно было бы также добавить функции потерь на выходы других промежуточных слоёв, однако они имееют следующие размеры:  
- (batch_size, n_feat, conv_out_frequency * cnn_out_channels) после свёртки
- (batch_size, n_feat, hidden_size) после GRU
- (batch_size, hidden_size) после attention

Учитывая, что для значительного сжатия и ускорения придётся уменьшать cnn_out_channels и hidden_size, то все перечисленные размеры уменьшатся становится трудно выбрать полезную функцию потерь. Для cross_entropy между предсказанными коэффициентами attention достаточно чтобы длины последовательностей были одинаковы, то есть не менять kernel_size[1] и stride[1]

### Quantization
Использовалась динамическая квантизация для сжатия GRU и Linear модулей. Как оказалось сжатие требует хранения некоторого количества информации о параметрах квантизации, поэтому при небольшом числе параметров например у Linear размер модели может увеличиться.  
По итогу, лучшая модель получена квантизацией только GRU и достигает сжатия в ~10.11 раз и ускорения в ~9.65 (также как и у модели без квантизации)

### Результаты
Как можно видеть среди побивших бейзлайн в 5.5e-5 лучшая скорость у модели с Dark Knowledge Distillation, затем добавляя к ней динамическую квантизацию GRU получается модель также с лучшим сжатием.
![изображение](https://user-images.githubusercontent.com/59803738/200187464-538263ae-8ba8-4f42-82a6-ec4a3ac807b4.png)


