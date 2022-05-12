## Onboarding

[Ссылка на админку Onboarding](https://devsrv10.dev.63pokupki.ru/admin/onboarding#/)

Логика работы:

- Ондобрдинг показывается при первом заходе в корзину, далее, на бэк улетает токен, который проверяется при следующем входе в корзину. Если токен есть, онбординг не запускается.
- Чтобы тестировать, необходимо залогинится под Маркетинг на деве. Далее, перевести шаблон в редактирование. Для этого, в списке страниц слева, выбрать необходимую страницу и справа от инпута "Идентификатор страницы" нажать на кнопку с карандашом.
- Есть кэш
- Токен хранится сутки, потом онбординг снова показывается один раз
- Переход от шага к шагу происходит по селекторам(класс, индентификатор) на странице.
- Чтобы определить правильно селектор на который будет переходить следующий шаг , можно использовать атрибут ID или специальный класс типа onboarding_cart_step_1, где 1 соответственно номер шага, этот момент нужно обговаривать с маркетингом.

### Подключение

1. В папке Common создаём папку Onboarding, в ней создаём файл Onboarding.ts(src/common/Onboarding/Onboarding.ts)
~~~
const introJs = require('@63pokupki/onboarding-core');
import axios from 'axios';
import { uuid } from 'uuidv4';
import { P63OnboardingStepI, IOnboardingEvents } from './OnboardingI';
import * as config from '@/config/MainConfig';
import { OnboardingNoAuthR } from '@/ifc/core/OnboardingNoAuthR';

/**
 * Подсказки по сайту для пользователей
 */
class Onboarding {
    /**
     * Стандартные настройки
     */
    public options = {
        tooltipClass: 'onboarding-base-steps',
        skipLabel: 'Пропустить',
        doneLabel: 'Закончить',
        nextLabel: 'Далее',
        hidePrev: true,
        hideNext: true,
        showProgress: true,
        showBullets: false,
        showStepNumbers: false,
        scrollTo: 'tooltip',
        disableInteraction: true,
        exitOnOverlayClick: false,
        touchpoint: 1024,
    };

    /**
     * Список шагов для сценария
     */
    public steps;

    /**
     * Идентификатор человека
     */
    public uuid;

    /**
     * Ключ идентификатора человека в localstorage
     */
    public uuidLocalStorageKey: string = 'onboarding_uuid';

    /**
     * Идентификатор готовности
     */
    public isReady: boolean = true;

    /**
     * Возможные ошибки
     */
    private _errors = {
        enviromentIsNotExist: 'Окружение исполнения не доступно',
        aliasIsNotExist: 'Идентификатор страницы не передан',
        scenarioIsNotExist: 'Сценарий для данной страницы отсутствует',
    };

    private intro;

    private scrollTo: Function;

    constructor(public alias: string, public events?: IOnboardingEvents) {
        this.intro = new introJs();
        this.scrollTo = scrollTo;
    }

    /**
     * Запуск сценария, основная функция
     */
    public play() {
        try {
            this.intro.start();

            this.initCloseButton();

            setTimeout(() => this.scrollTo(), 500);

            this.setOverflowProperty();
        } catch (e) {
            console.error(e);
        }
    }

    /**
     * Инициализация
     */
    public async init() {
        try {
            // проверка на готовность
            if (!this.isEnviromentExist()) {
                throw new Error(this._errors.enviromentIsNotExist);
            }

            // проверка на идентификатор страницы
            if (!this.alias) {
                throw new Error(this._errors.aliasIsNotExist);
            }

            // получение uuid
            this.getUUID();

            // получение сценария с сервера
            await this.getStepsView();

            // сценария нет
            if (!this.steps) {
                console.log(this._errors.scenarioIsNotExist);
                return;
            }

            this.intro = this.fUpdatePropertiesStepByStepHook();

            return true;
        } catch (e) {
            console.error(e);
            return false;
        }
    }

    /**
     * Получение и установка uuid
     */
    private getUUID() {
        try {
            // uuid существует
            const existedId = localStorage.getItem(this.uuidLocalStorageKey);
            if (existedId) {
                this.uuid = existedId;
                return existedId;
            }

            // uuid не существует, генерируем
            const id = uuid();
            localStorage.setItem(this.uuidLocalStorageKey, id);
            this.uuid = id;
            return id;
        } catch (e) {
            console.error(e);
        }
    }

    /**
     * Установка uuid
     */
    private setUUID(id) {
        try {
            localStorage.setItem(this.uuidLocalStorageKey, id);
        } catch (e) {
            console.error(e);
        }
    }

    /**
     * Проверка на доступность среды
     */
    private isEnviromentExist() {
        if (window && window.document) {
            return true;
        }

        return false;
    }

    /**
     * Получить шаги для отображения пользователю
     */
    private async getStepsView() {
        try {
            const options = {
                obkey: this.uuid,
                alias: this.alias,
            };

            // const {
            //     data: { is_active, list_m_step, list_d_step, obkey },
            // }
            let is_active = null;
            let list_m_step = null;
            let list_d_step = null;
            let obkey = null;
            await axios
                .post(config.coreApi.baseURL + OnboardingNoAuthR.listStep.route, options)
                .then(function(response) {
                    is_active = response.data.data.is_active;
                    list_m_step = response.data.data.list_m_step;
                    list_d_step = response.data.data.list_d_step;
                    obkey = response.data.data.obkey;
                })
                .catch(function(error) {
                    console.log(error);
                });

            // переназначаем uuid в случае если пользователь сменился (уменьшает кол-во запросов к бд на беке)
            if (obkey) {
                this.setUUID(obkey);
            }

            // если сценарий не должен запуститься ничего не возвращаем
            if (!is_active) {
                return;
            }

            let steps = null;

            // определение ширины устройства
            if (this.isTouchDevice()) {
                if (list_m_step.length == 0) {
                    return;
                }
                steps = this.transformStepsForLibrary(list_m_step);
            } else {
                if (list_d_step.length == 0) {
                    return;
                }
                steps = this.transformStepsForLibrary(list_d_step);
            }

            this.steps = steps;

            return steps;
        } catch (e) {}
    }

    private transformStepsForLibrary(steps: P63OnboardingStepI[]) {
        try {
            const sorted_steps = steps.sort((a, b) => a.step_number - b.step_number);

            return sorted_steps.map(step => {
                return {
                    intro: step.text || 'Текст отсутствует',
                    element: step.selector ? this.getElement(step.selector) : null,
                    tooltipClass: step.tpl || this.options.tooltipClass,
                    skipLabel: step.btn_skip || this.options.skipLabel,
                    doneLabel: step.btn_end || this.options.doneLabel,
                    nextLabel: step.btn_next || this.options.nextLabel,
                    beforeStep: step.event_before,
                    afterStep: step.event_after,
                };
            });
        } catch (e) {
            console.error(e);
        }
    }

    /**
     * Определение режима показа - мобильные/десктоп
     */
    private isTouchDevice() {
        try {
            const w = document.documentElement.clientWidth;

            if (w <= this.options.touchpoint) {
                return true;
            } else {
                return false;
            }
        } catch (e) {
            console.error(e);
        }
    }

    /**
     * Проверка видимости (существования) элемента
     * @param {} element - элемент
     */
    private isElementExist(element) {
        try {
            if (!element) {
                throw new Error('Элемент не передан');
            }

            const { height, width } = element.getBoundingClientRect();

            if (height !== 0 && width !== 0) {
                return true;
            } else {
                return false;
            }
        } catch (e) {
            console.error(e);
        }
    }

    /**
     * Получение видимого элемента по css селектору
     * @param {String} selector - css валидный селектор
     */
    private getElement(selector) {
        try {
            if (!selector) {
                throw new Error('Селектор не передан');
            }

            const elements = Array.prototype.slice.call(document.querySelectorAll(selector));

            const visibleElement = elements.find(el => this.isElementExist(el));

            if (!visibleElement) {
                console.log('Элемент: ' + selector + ' не найден');
                return;
            }

            return visibleElement;
        } catch (e) {
            console.error(e);
        }
    }

    /**
     * Обновление настроек под каждый шаг подсказок, смена текста, оформления и тд
     */
    private fUpdatePropertiesStepByStepHook() {
        try {
            const self = this;
            // установка стандартных настроек
            self.intro.setOptions({
                ...self.options,
                steps: self.steps,
            });

            // переопределение настроек под каждый шаг
            self.intro.onbeforechange(() => {
                let step = self.steps[self.intro._currentStep];

                self.intro.setOptions({
                    ...self.options,
                    nextLabel: step.nextLabel || self.options.nextLabel,
                    doneLabel: step.doneLabel || self.options.doneLabel,
                    skipLabel: step.skipLabel || self.options.skipLabel,
                    tooltipClass: step.tooltipClass || self.options.tooltipClass,
                });

                // хук для события перед показом шага
                if (self.events && step.beforeStep) {
                    if (self.events.hasOwnProperty(step.beforeStep)) {
                        self.events[step.beforeStep]();
                    }
                }
            });

            // переопределение настроек под каждый шаг
            self.intro.onafterchange(() => {
                let step = self.steps[self.intro._currentStep];

                // хук для события после показа шага
                if (self.events && step.afterStep) {
                    if (self.events.hasOwnProperty(step.afterStep)) {
                        self.events[step.afterStep]();
                    }
                }
            });

            return self.intro;
        } catch (e) {
            console.error(e);
        }
    }

    /**
     * Инициализация кнопки Закрыть
     */
    private initCloseButton() {
        try {
            let el = document.querySelector('.introjs-tooltip');

            // создание иконки крестика
            let icon = document.createElement('i');
            icon.className = 'ds-icon icon-close';

            // создание кнопки
            let close = document.createElement('button');
            close.append(icon);
            close.className = `introjs-tooltip__close`;
            close.addEventListener('click', () => this.intro.exit(true));

            // добавление кнопки
            if (el) {
                el.append(close);
            }
        } catch (e) {
            console.error(e);
        }
    }

    private setOverflowProperty() {
        try {
            let el = document.querySelector('.introjs-tooltip');

            if (!el || !this.isTouchDevice()) {
                return;
            }

            el.scrollTop = 0;
        } catch (e) {
            console.error(e);
        }
    }
}

/**
 * Прокрутка
 * @param {Number} x - координата X прокрутки
 * @param {Number} y - координата Y прокрутки
 */
export const scrollTo = (x: number = 0, y: number = 0) => {
    try {
        const options: ScrollToOptions = {
            top: x,
            left: y,
            behavior: 'smooth',
        };

        setTimeout(() => {
            window.scrollTo(options);
        }, 25);
    } catch {
        window.scrollTo(x, y);
    }
};

export default Onboarding;
~~~
2. Далее, там же, создаём файл OnboardingI.ts(src/common/Onboarding/OnboardingI.ts):
~~~
/**
 * Набор событий, котоыре могут быть выполнены перед/после шага
 */
export interface IOnboardingEvents {
    [event: string]: Function;
}

/**
 * Интерфейс таблицы Onboarding - шаги подсказок на странице
 */
export interface P63OnboardingStepI {
    id?: number; // ID
    onboarding_page_id?: number; // ID страницы
    name?: string; // понятное имя шага - например welcome / 1 шаг
    text?: string; // html - текст шага
    step_number?: number; // номер шага
    tpl?: string; // шаблон - внешний вид шага - css селектор
    selector?: string; // идентификатор класса или id елемена на странице на который навешан шаг
    type?: string; // шаг цепочки версии (мобильной или десктопной
    btn_view?: string; // текст кнопки просмотр
    btn_next?: string; // текст кнопки далее
    btn_skip?: string; // текст кнопки пропустить
    btn_end?: string; // текст кнопки закрыть',
    event_before?: string; // JS событие срабатывающее до отображения шага
    event_after?: string; // JS событие срабатывающее после скрытия шага
}

~~~

3. Добавляем роуты src/ifc/core/OnboardingAdminR.ts

~~~

import { P63OnboardingPageIxI, P63OnboardingPageI } from './EntitySQL/P63OnboardingPageE';
import { P63OnboardingStepI } from './EntitySQL/P63OnboardingStepE';
import { P63OnboardingTplI } from './EntitySQL/P63OnboardingTplE';

/**
 * Онбординг модуль администратора
 */
export namespace OnboardingAdminR {
    export const Alias = 'OnboardingAdmin';

    // =======================================================
    /** Получить список страниц */
    export namespace listPage {

        /** APIURL */
        export const route = '/onboarding-admin/list-page';

        /** Alias действия */
        export const action = 'list-page';

        /** Параметры api запроса */
        export interface RequestI {
        }

        /** Параметры api ответа */
        export interface ResponseI {
            list_page: P63OnboardingPageI[];
        }
    }

    // =======================================================
    /** цепочку подсказок страницы */
    export namespace onePage {

        /** APIURL */
        export const route = '/onboarding-admin/one-page';

        /** Alias действия */
        export const action = 'one-page';

        /** Параметры api запроса */
        export interface RequestI {
            onboarding_page_id: number;
        }

        /** Параметры api ответа */
        export interface ResponseI {
            one_page: P63OnboardingPageIxI;
        }
    }

    // =======================================================
    /** Добавить страницу */
    export namespace addPage {

        /** APIURL */
        export const route = '/onboarding-admin/add-page';

        /** Alias действия */
        export const action = 'add-page';

        /** Параметры api запроса */
        export interface RequestI {
            name: string; // Понятно имя страницы
            alias: string; //  Псевдоним страницы
        }

        /** Параметры api ответа */
        export interface ResponseI {
            onboarding_page_id: number;
            one_page: P63OnboardingPageI;
        }
    }

    // =======================================================
    /** Создать страницу черновик */
    export namespace addPageTest {

        /** APIURL */
        export const route = '/onboarding-admin/add-page-test';

        /** Alias действия */
        export const action = 'add-page-test';

        /** Параметры api запроса */
        export interface RequestI {
            alias: string; //  Псевдоним страницы
        }

        /** Параметры api ответа */
        export interface ResponseI {
            onboarding_page_id: number;
            one_page: P63OnboardingPageI;
        }
    }

    // =======================================================
    /** Сделать тестовую страницу по умолчанию */
    export namespace makePageTestByDefault {

        /** APIURL */
        export const route = '/onboarding-admin/make-page-test-by-default';

        /** Alias действия */
        export const action = 'make-page-test-by-default';

        /** Параметры api запроса */
        export interface RequestI {
            alias: string; //  Псевдоним страницы
        }

        /** Параметры api ответа */
        export interface ResponseI {
            one_page: P63OnboardingPageIxI;
        }
    }

    // =======================================================
    /** Сохранить основные параметры цепочки */
    export namespace savePage {

        /** APIURL */
        export const route = '/onboarding-admin/save-page';

        /** Alias действия */
        export const action = 'save-page';

        /** Параметры api запроса */
        export interface RequestI {
            onboarding_page_id: number; // ID По которуму сохраняется страница
            name?: string; // понятное имя страницы
            descript?: string; // Описание
        }

        /** Параметры api ответа */
        export interface ResponseI {
            one_page: P63OnboardingPageI;
        }
    }

    // =======================================================
    /** Удалить страницу */
    export namespace delPage {

        /** APIURL */
        export const route = '/onboarding-admin/del-page';

        /** Alias действия */
        export const action = 'del-page';

        /** Параметры api запроса */
        export interface RequestI {
            alias: string;
        }

        /** Параметры api ответа */
        export interface ResponseI {
            one_page: P63OnboardingPageI;
        }
    }

    // =======================================================
    /** Удалить тестовую цепочку */
    export namespace delPageTest {

        /** APIURL */
        export const route = '/onboarding-admin/del-page-test';

        /** Alias действия */
        export const action = 'del-page-test';

        /** Параметры api запроса */
        export interface RequestI {
            alias: string;
        }

        /** Параметры api ответа */
        export interface ResponseI {
            one_page: P63OnboardingPageI;
        }
    }

    // =======================================================
    /** Получить список шагов цепочки */
    export namespace listStep {

        /** APIURL */
        export const route = '/onboarding-admin/list-step';

        /** Alias действия */
        export const action = 'list-step';

        /** Параметры api запроса */
        export interface RequestI {
            alias: string; // Псевдоним страницы
            is_test: boolean; // тестовая цепочка?
            type: string; // mobile|desktop
        }

        /** Параметры api ответа */
        export interface ResponseI {
            list_step: P63OnboardingStepI[];
        }
    }

    // =======================================================
    /** добавить шаблон шаблонов */
    export namespace addTpl {

        /** APIURL */
        export const route = '/onboarding-admin/add-tpl';

        /** Alias действия */
        export const action = 'add-tpl';

        /** Параметры api запроса */
        export interface RequestI {
            name:string; // Название шаблона
            tpl:string; // CSS селектор для шаблона в дизайн системе
        }

        /** Параметры api ответа */
        export interface ResponseI {
            onboarding_tpl_id: number;
            one_onboarding_tpl: P63OnboardingTplI;
        }
    }

    // =======================================================
    /** сохранить шаблон */
    export namespace saveTpl {

        /** APIURL */
        export const route = '/onboarding-admin/save-tpl';

        /** Alias действия */
        export const action = 'save-tpl';

        /** Параметры api запроса */
        export interface RequestI {
            onboarding_tpl_id: number;
            name:string; // Название шаблона
            tpl:string; // CSS селектор для шаблона в дизайн системе
        }

        /** Параметры api ответа */
        export interface ResponseI {
            one_onboarding_tpl: P63OnboardingTplI;
        }
    }

    // =======================================================
    /** Получить список шаблонов */
    export namespace listTpl {

        /** APIURL */
        export const route = '/onboarding-admin/list-tpl';

        /** Alias действия */
        export const action = 'list-tpl';

        /** Параметры api запроса */
        export interface RequestI {
        }

        /** Параметры api ответа */
        export interface ResponseI {
            list_tpl:P63OnboardingTplI[];
        }
    }

    // =======================================================
    /** удалить шаблон */
    export namespace delTpl {

        /** APIURL */
        export const route = '/onboarding-admin/del-tpl';

        /** Alias действия */
        export const action = 'del-tpl';

        /** Параметры api запроса */
        export interface RequestI {
            onboarding_tpl_id:number;
        }

        /** Параметры api ответа */
        export interface ResponseI {
        }
    }

    // =======================================================
    /** Добавить шаг цепочки */
    export namespace addStep {

        /** APIURL */
        export const route = '/onboarding-admin/add-step';

        /** Alias действия */
        export const action = 'add-step';

        /** Параметры api запроса */
        export interface RequestI {
            onboarding_page_id: number;
            name: string; // Имя шага
            step_number: number; // Номер шага
            type: string; // mobile|desktop
        }

        /** Параметры api ответа */
        export interface ResponseI {
            step_id: number;
            one_step: P63OnboardingStepI;
        }
    }

    // =======================================================
    /** Удалить шаг цепочки */
    export namespace delStep {

        /** APIURL */
        export const route = '/onboarding-admin/del-step';

        /** Alias действия */
        export const action = 'del-step';

        /** Параметры api запроса */
        export interface RequestI {
            onboarding_step_id: number;
        }

        /** Параметры api ответа */
        export interface ResponseI {
        }
    }

    // =======================================================
    /** Сохранить шаг цепочки */
    export namespace saveStep {

        /** APIURL */
        export const route = '/onboarding-admin/save-step';

        /** Alias действия */
        export const action = 'save-step';

        /** Параметры api запроса */
        export interface RequestI {
            onboarding_step_id: number; // ID шага по которому сохраняется запись
            name?: string; // понятное имя шага - например welcome / 1 шаг
            text?: string; // html - текст шага
            step_number?: number; // номер шага
            event_before?: string; // JS событие срабатывающее до отображения шага
            event_after?: string; // JS событие срабатывающее после скрытия шага
            onboarding_tpl_id?: number; // шаблон - внешний вид шага - css селектор
            selector?: string; // идентификатор класса или id елемена на странице на который навешан шаг
            type?: string; // шаг цепочки версии (мобильной или десктопной
            btn_view?: string; // текст кнопки просмотр
            btn_next?: string; // текст кнопки далее
            btn_skip?: string; // текст кнопки пропустить
            btn_end?: string;// текст кнопки закрыть',
        }

        /** Параметры api ответа */
        export interface ResponseI {
            one_step: P63OnboardingStepI;
        }
    }

}

~~~

///
- src/ifc/core/OnboardingNoAuthR.ts
~~~

import { P63OnboardingPageIxI, P63OnboardingPageI } from './EntitySQL/P63OnboardingPageE';
import { P63OnboardingStepI } from './EntitySQL/P63OnboardingStepE';

/**
 * Онбоардинг модуль пользователя
 */
export namespace OnboardingNoAuthR {
    export const Alias = 'OnboardingNoAuth';

    // =======================================================
    /** Получить список шагов цепочки */
    export namespace listStep {

        /** APIURL */
        export const route = '/onboarding-no-auth/list-step';

        /** Alias действия */
        export const action = 'list-step';

        /** Параметры api запроса */
        export interface RequestI {
            obkey: string; // onboarding key
            alias: string; // Псевдоним страницы
        }

        /** Параметры api ответа */
        export interface ResponseI {
            is_active: boolean; // Показывать или нет
            obkey: string; // Фронтовый или новый токен если фронтовый не актуальный
            list_m_step: P63OnboardingStepI[]; // Шаги для мобильной версии
            list_d_step: P63OnboardingStepI[]; // Шаги для десктопной версии
        }
    }

    // =======================================================
    /** Инкрементировать действие пользователя */
    export namespace incrAction {

        /** APIURL */
        export const route = '/onboarding-no-auth/incr-action';

        /** Alias действия */
        export const action = 'incr-action';

        /** Параметры api запроса */
        export interface RequestI {
            obkey: string; // onboarding key
            alias: string; // Псевдоним страницы
            next?: boolean; // 0/1 - Продолжить
            skip?: boolean; // 0/1 - Пропустить
            view?: boolean; // 0/1 - Посмотреть
            end?: boolean; // 0/1 - Закрыть
        }

        /** Параметры api ответа */
        export interface ResponseI {
        }
    }

}

~~~
///


4. Интерфейсы
- src/ifc/core/EntitySQL/P63OnboardingPageE.ts
~~~

/**
 * Интерфейс таблицы Onboarding - подсказок на странице
 */
export interface P63OnboardingPageIxI {
    id?: number; // ID
    name?: string; // понятное имя страницы
    alias?: string; // Ключ страницы
    is_test?: boolean; // Тестовая цепочка?
}

/**
 * Интерфейс таблицы Onboarding - подсказок на странице
 */
export interface P63OnboardingPageI {
    id?: number; // ID
    name?: string; // понятное имя страницы
    alias?: string; // Ключ страницы
    descript?: string; // Описание
    is_test?: boolean; // Тестовая цепочка?
}

~~~
///

Интерфейсы шагов
- src/ifc/core/EntitySQL/P63OnboardingStepE.ts
~~~

/**
 * Интерфейс таблицы Onboarding - шаги подсказок на странице
 */
export interface P63OnboardingStepI {
    id?: number; // ID
    onboarding_page_id?: number; // ID страницы
    name?: string; // понятное имя шага - например welcome / 1 шаг
    text?: string; // html - текст шага
    step_number?: number; // номер шага
    event_before?: string; // JS событие срабатывающее до отображения шага
    event_after?: string; // JS событие срабатывающее после скрытия шага
    onboarding_tpl_id?: number; // шаблон - внешний вид шага - css селектор
    selector?: string; // идентификатор класса или id елемена на странице на который навешан шаг
    type?: string; // шаг цепочки версии (мобильной или десктопной
    btn_view?: string; // текст кнопки просмотр
    btn_next?: string; // текст кнопки далее
    btn_skip?: string; // текст кнопки пропустить
    btn_end?: string;// текст кнопки закрыть',
}


~~~

///

Интерфейсы токена
- src/ifc/core/EntitySQL/P63OnboardingTokenE.ts
~~~
/**
 * Интерфейс таблицы Onboarding - подсказок на странице
 */
export interface P63OnboardingTokenIxI {
    id?: number; // ID
    alias?: string; // Ключ страницы
    token?: string; // Тестовая цепочка?
    user_id?: number; // ID пользователя
}

/**
 * Интерфейс таблицы Onboarding - подсказок на странице
 */
export interface P63OnboardingTokenI {
    id?: number; // ID
    alias?: string; // Псевдоним страницы для onboarding
    token?: string; // obkey - токен для onboarding
    user_id?: number; // ID пользователя
    show_counter?: number; // Счетчик показов
    view?: number; // Счетчик - Просмотренно
    next?: number; // Счетчик - Далее
    skip?: number; // Счетчик - Пропустить
    end?: number; // Счетчик - Закрыть
    created_at?: string; // Дата создания
    updated_at?: string; // Дата обновления
}
~~~

Интерфейс таблицы
- src/ifc/core/EntitySQL/P63OnboardingTplE.ts
~~~
/**
 * Интерфейс таблицы Onboarding - подсказок на странице
 */
export interface P63OnboardingTplI {
    id?: number; // ID
    name?: string; // Имя шаблона
    tpl?: string; // шаблон
}
~~~
///

Инициализация

- Необходимо подключить библиотеку @63pokupki/onboarding-core
- Лучшее место для подключения подсказок, при инициализации страницы в методе init в контролере
- Пример инициализации подсказок(проект client.front src/pages/cart/view/ctrl_cart.ts):
~~~
...
import Onboarding from '@/common/Onboarding/Onboarding';
import { PurchaseStatusT } from '@/ifc/core/EntitySQL/PurchaseE';

export class CartCtrl extends BaseCtrl {
    public conf = conf;

    public status: CartStoreI.Status = gVuexSys.registerModuleStatus(new CartStoreI.Status());
    public error: CartStoreI.Error = gVuexSys.registerModuleError(new CartStoreI.Error());
    public list: CartStoreI.List = gVuexSys.registerModuleList(new CartStoreI.List());
    public tree: CartStoreI.Tree = gVuexSys.registerModuleTree(new CartStoreI.Tree());
    public cmd: CartStoreI.Cmd = gVuexSys.registerModuleCmd(new CartStoreI.Cmd());
    public one: CartStoreI.One = gVuexSys.registerModuleOne(new CartStoreI.One());
    public ix: CartStoreI.Ix = gVuexSys.registerModuleIx(new CartStoreI.Ix());

    private querySys: QuerySys = null;

    constructor(vuexSys: VuexSys) {
        super(vuexSys);

        this.querySys = new QuerySys();
        this.querySys.fConfig(conf.coreApi);
    }

    /** Инициализация страницы */
    public async fInit() {
        if (this.status.curr_tab === CartN.TabsT.active) {
            this.fGetListActiveInvoice()

        } else {
            this.fGetListArchiveInvoice();
        }

        const onboarding = new Onboarding('onboarding_cart_page');
        if (this.status.cart_owner_user_id ) {
            this.fGetUserData();
        }
        await Promise.all([
			this.fGetListFavoriteStockID(),
            // Получить баннеры
            this.fGetListBanner(),
            // Получить топ подборки
            this.fGetListTopSale(),
            this.fGetListTopItemByCategory(),
            this.fGetListTopItemByStock(),
            // Получить организаторов для поиска
            this.fGetListOrgUser(),
            onboarding.init()
		])
        onboarding.play();
    }
    ...
~~~
Импортируем в инит файле
- src/pages/cart/view/init_cart.ts
~~~
import "@63pokupki/onboarding-core/index.css"
import "@63pokupki/onboarding-core/index.js"
~~~

