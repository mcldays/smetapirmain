# `@rnstream/pirmainDivide`

> Элементы управления @rnstream/smetapir


## Табличная форма отображения 
Основной компонент для Сметы ПИР 

### Использование компонента
1. Подключить и зарегистрировать компонент:
```
import {PirMain, PirMainDivide, PirMainTOR, TableWithChilds} from "@rnstream/smetapir";
@Component({
	components:{
		PirMain,
		PirMainDivie,
		PirMainTOR,
		TableWithChilds
	}
})
```

2. Разместить компонент в шаблоне:
```
 <PirMainDivide 
             v-bind="displayFormProps"
             :options="options"
             v-on="$listeners"
             @editorPreviewRequest="handleEntityPreviewRequest"
             @notification="notificationHandler"/>
```

### Props

|Prop| Описание  | Тип |
|:-|:-|:-|
| **treeId** | Идентификатор визуального дерева | number |
| **data** | Объект состоящий из nodePath, entityEditorData и displayType | TabViewModel |
| **nodePath** | Выбранный узел дерева | VisualTreeNodePathModel[] |
| **entityController** | Контроллер для сущностей | EntityController |
| **visualTreeController** | Контроллер для доступа к визуальным деревьям | VisualTreeController |
| **fileController** | Контроллер для работы с файлами   | FileController |
| **workflowService** | - | IWorkflowService |
| **entityTypePicker** | - | IEntityTypePicker |
| **entityEditor** | - | IEntityEditor |

### Events

|Event| Описание  | Тип |
|:-|:-|:-|
| **rowClick** | Клик ЛКМ на строку | any (присутствуют следующие обязательные поля: EntityId, EntityName, EntityTypeId)|
| **entityCreated** | Создание сущности | any |
| **navigate** | Находим сущность в дереве | (присутствуют следующие обязательные поля: nodePath, entityTypeNode) |

#### data (TabViewModel)
```
export class TabViewModel {

	constructor(displayType: DisplayTypeEnum, nodePath: VisualTreeNodePathModel[], entityEditorData?: EntityEditorData) {
		this.nodePath = nodePath;
		this.entityEditorData = entityEditorData;
		this.DisplayType = displayType;
	}
```

#### displayType (DisplayTypeEnum)
```
export enum DisplayTypeEnum {
	None = 0,

	EntityForm = 1,

	TableForm = 2,

	GanttGraph = 3,

	RelatedEntityForm = 4,

	PeerReview = 5,

	VersionedEntityForm = 6,

	GalleryForm = 7,

	ComplexTableForm = 8
}
```

#### Интерфейс IWorkflowService
```
export interface IWorkflowService {
	GetWorkflows(entityTypeId: number): Promise<IWorkflowModel[]>;

	Start(workflowId: number, entityId: number): AxiosPromise;

	GetActions(entityModel: EntityMasterModel): RnToolbarItem[];
}
```

#### Интерфейс IEntityTypePicker
```
export interface IEntityTypePicker {
	(treeId: number, nodePath: VisualTreeNodePathModel[]): Promise<EntityTypeNodeModel>
}
```

#### Интерфейс IEntityEditor
```
export interface IEntityEditor {
	(data: EntityEditorData): Promise<EntityModel>
}
```
