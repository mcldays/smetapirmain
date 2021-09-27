<template>
  <div class="smetaForm">
    <RnToolbar ref="slotTableToolbar"/>
    <EntityTableForm
        v-if="nodePath.length > 0"
        :treeId="treeId"
        :nodePath="nodePath"
        :entityController="entityController"
        :visualTreeController="visualTreeController"
        :fileController="fileController"
        :entity-type-picker="entityTypePicker"
        :isHideToolbar="false"
        :isHideStatusbar="false"
        :is-hide-folders="true"
        :cascade-max-depth="5"
        :workflowService="workflowService"
        :edit-mode="EditMode.InlineEditing"
        :cascade-mode="2"
        :cascade="true"
        @navigate="emitNavigate"
        @rowClick="rowclick"
        style="--viewport-height: 100%;--toolbar-height: 0px;"
        ref="mainTable"
    />
    <div style="display: flex; padding: 10px 0px 0px 5px;"
         v-if="this.preloaderState">
      <v-progress-circular
          indeterminate
          color="amber"
      ></v-progress-circular>
      <div class="grey--text mb-2" style="margin-left: 15px;">
        Ваша расценка добавляется, пожалуйста, подождите...
      </div>
    </div>

    <VDialogWrapper v-model="treeDialogOpened" :title="$t('Выберите строку справочника работ')" :width = 1600 scrollable>
      <MultiSelectEntityTreeView
          v-if="treeDialogOpened"
          :treeId="519"
          :visualTreeController="visualTreeController"
          @input="multiselectValuesChange"
          :value="getSelectedValuesForTree()"
          :connectionGroupId="connectionGroupId"
          :multipleConnection="false"
          :targetEntityTypeId="entityTypeId"
      />
      <!--          :targetEntityTypeId="entityTypeId"-->
      <template v-slot:actionButton>
        <v-btn text @click="applySelectedValuesFromTree">{{ $t('Save') }}</v-btn>
      </template>
    </VDialogWrapper>

    <v-dialog
        persistent
        v-model="this.dialog">
      <v-card>
        <v-card-title>
          Добавить строку
        </v-card-title>
    <GlobalDisplayForm :options="formOptions"
                       style="flex-basis: 100%; overflow: hidden; padding: 20px"
                       @entityCreated="entityCreated"
                       @entityUpdated="entityUpdated"/>
        <v-card-actions>
          <v-spacer></v-spacer>
          <v-btn
              color="primary"
              text
              @click="dialog = false"
          >
            Закрыть
          </v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>

  </div>

</template>

<script lang = "ts">
import {Component, Prop, Vue, Watch,} from "vue-property-decorator";
import {
  EntityConnectionValue,
  EntityEditorData,
  EntityTypeNodeModel,
  FileController,
  IEntityEditor,
  IEntityEditorWindowComponent,
  IEntityTypePicker,
  IWorkflowService,
  RnConnectionInput,
  RnToolbar,
  RnToolbarGroup,
  VisualTreeNodePathModel,
  IdNameWarningModel,
  IIdNameNodePathModel,
  IEntityController,
  PageModel,
  VisualTreeController,
  EntityModel,
  EntityMasterModel,
  ConnectionReadModel,
  ConnectionValueModel,
  IdNameModel,
  IdNameTypeModel,
  ConnectionValueEntity,
  EntityCreatedEventArg,
  EntityEditorInfoObject,
  Entity_AttributeValueModel,
  AttributeEditorModel,
  VisualTreeNodeQueryModel,
  VisualTreeNodeQueryParameter,
  GlobalDisplayFormParams, DisplayTypeEnum,

} from "@rnstream/ui";
import VDialogWrapper from '@/components/Common/VDialogWrapper/VDialogWrapper.vue';
import Axios, {AxiosPromise, AxiosResponse} from "axios";
import Clonning from "@rnstream/ui/src/helpers/Clonning";
import MultiSelectEntityTreeView from "@rnstream/ui/src/inputs/controls/MultiSelectTreeView/MultiSelectEntityTreeView.vue"
import ConnectionInput from "@rnstream/ui/src/inputs/controls/ConnectionInput/ConnectionInput.vue"
import {AttributeEditorHelper} from "@rnstream/ui/src/entityeditor/components/AttributeEditorHelper";
import {EntityController} from '@/controllers/EntityController';

import {SiteUrl} from "@/components/Settings/SiteUrl";
import {EntityTableForm } from "@rnstream/tableentityform";
import {EditMode} from '@rnstream/tableentityform/src/data/enums/EditMode';
import {VisualFormActionReadData} from "@/models/generated/api.generated.clients";
@Component({
  name: "PirMainTOR",
  components: {
    RnToolbar,
    VDialogWrapper,
    MultiSelectEntityTreeView,
    ConnectionInput,
    RnConnectionInput,
    EntityTableForm
  }
})
export default class PirMainTOR extends Vue {

  //--------------- props

  @Prop({ type: Number }) treeId: number;

  @Prop()
  value: VisualFormActionReadData;

  private localValue: VisualFormActionReadData = {
    Id: null,
    Name: null,
    Caption: null,
    ActionType: null,
    VisualFormId: null,
    ActionInterface: null,
    IconName: null
  } as VisualFormActionReadData;

  // Путь до конкректного элемента/папки дерева в левой части РН-СТРИМ
  @Prop() nodePath: VisualTreeNodePathModel[];

  // Контроллер для доступа к визуальным деревьям и методам взаимодействия с ними
  @Prop() visualTreeController: VisualTreeController;

  // Контроллер для дотсупа к сущностям и методам взаимодействия с ними
  @Prop() entityController: IEntityController;

  // Контроллер для доступа к файлам (наверное нужен для экспорта и вкладывания документов хз)
  @Prop() fileController: FileController;

  // Интерфейс, который судя по всему определяет какой тип сущности мы подвергаем каким то actions
  @Prop() entityTypePicker: IEntityTypePicker;

  // Интерфейс, который судя по всему открывает диалог редактирования сущности, когда EntityType уже известен
  @Prop() entityEditor: IEntityEditor;

  // Какая то херня с потоками, нужно для корректного сохранения результата экшна в самом дереве РН-СТРИМ (вроде как)
  @Prop() workflowService: IWorkflowService;

  // Признак видимости тулбара таблицы
  @Prop() isHideToolbar: boolean;

  // триггеры для nodePathChanged
  private EditMode = EditMode
  private tableNodeCreateAllowed: boolean = true;
  private selectedEntityData: EntityEditorData = null;
  private selectedEntityName: string = "";
  private tableNodeHidden: boolean = false;
  private tableNodeEditAllowed: boolean = true;
  private tableRowNodeHidden: boolean = false;
  private entityTypeNode :any ;
  private dialog: boolean = false;
  private buttonState : boolean = false;
  private TORDepthOne : VisualTreeNodePathModel[];
  private TORDepthTwo : VisualTreeNodePathModel[];
  private TORDepthThree : VisualTreeNodePathModel[];
  private TORDepthFour : VisualTreeNodePathModel[];
  private TORNodePathOnString : VisualTreeNodePathModel[];
  private TORNodePathOnSection : VisualTreeNodePathModel[]
  private preloaderState : boolean = false;
  private SectionList : string[] = [
    "Строка сметы по форме 2П",
    "Строка сметы на ИЭИ",
    "Строка сметы на ОНЗТ",
    "Строка сметы по форме 3П",
    "Строка календарного плана",
    "Строка коммерческого предложения",
    "Строка сметы на ИКИ"
  ]
  private TotalSectionList : string[] = [
      `Строка "Итого" по разделу сметы (2П)`,
      `Строка "Итого" по разделу календарного плана`,
      `Строка "Итого" по разделу комм. предложения`,
      `Строка "Итого" по разделу сводной сметы`,
      `Строка "Итого" по разделу сметы ИИ по форме 2П`,
      `Строка "Итого" по разделу сметы ИКИ`,
      `Строка "Итого" по разделу сметы ИЭИ`,
      `Строка "Итого" по разделу сметы ОНЗТ`,
      `Строка "Итого" по разделу сметы по форме 3П`
  ]

  private TotalList : string[] = [
      `Строка "Итого"  календарного плана`,
      `Строка "Итого" по коммерческому предложению`,
      `Строка "Итого" сводной сметы`,
      `Строка "Итого" сметы (2П)`,
      `Строка "Итого" сметы ИИ по форме 2П`,
      `Строка "Итого" сметы ИКИ`,
      `Строка "Итого" сметы ИЭИ`,
      `Строка "Итого" сметы ОНЗТ`,
      `Строка "Итого" сметы по форме 3П`


  ]


  //возможные типы сущностей для ивентов тулбара
  private possibleEntityTypes: any


  entityCreated(data: any): any {
    this.dialog = false;
    return data;
  }


  emitNavigate(nodePath: VisualTreeNodePathModel[], entityTypeNode?: EntityTypeNodeModel) {
    this.$emit("navigate", nodePath, entityTypeNode);
  }

  // событие передачи кнопок и экшнов в тулбар
  toolbarActions(actions: any): void {

    (this.$refs.mainTable as any).generateToolbarButtons(actions);
    return actions;
  }

  // Биндим тулбар
  mounted() {
    this.mountMainToolbar();
  }

  // Кастомный тулбар
  async mountMainToolbar() {
    // Создаем группу кнопок для основной таблицы
    let mainTableToolbarGroup: RnToolbarGroup =
      new RnToolbarGroup("Управление расценками", "MainTableGroup", [
        {
          Name: "Добавить",
          Icon: "mdi-plus",
          Enabled: true,
          Action: null,
          Children: [
            {
              Name: this.$t("Факт. затраты") as string,
              Icon: "mdi-plus",
              Enabled: await this.canCreateFacts(),
              Action: this.factsAdd
            },
            {
              Name: this.$t("Команд-е расходы") as string,
              Icon: "mdi-plus",
              Enabled: await this.canCreateBussines(),
              Action: this.bussinesAdd
            },
            {
              Name: this.$t("РИУ ОИ") as string,
              Icon: "mdi-plus",
              Enabled: await this.canCreateRIU(),
              Action: this.estimatedStringAdd,
            }
          ]
        },
      ]);

    (this.$refs.mainTable as any).generateToolbarButtons(mainTableToolbarGroup);
  }

  //ивент для активации кнопок
  async canCreate() {
    if (!this.tableNodeCreateAllowed) {
      return false;
    }

    let entityTypes = await this.visualTreeController.GetEntityTypes(
        this.treeId,
        this.nodePath
    );

    return entityTypes && entityTypes.data.Data.length > 0;
  }


  async canCreateRIU(){
    let result = await this.getStringType();
   if(result == 1){
     return true
   }
   return false
  }

  async canCreateFacts(){
    let result = await this.getStringType();
    if(result == 2){
      return true
    }
    return false
  }

  async canCreateBussines(){
    let result = await this.getStringType();
    if(result == 3){
      return true
    }
    return false
  }


  //rowclick
  async rowclick(entityData : any): Promise<void> {

    switch (entityData.Depth){
      case 1: {

        this.TORDepthOne = this.$deepCopy(this.nodePath) as VisualTreeNodePathModel[]
        let model = await this.getNode(this.nodePath) as any
        this.TORDepthOne.push(new VisualTreeNodePathModel(model[0].NodeId, model[0].EntityId, null, null, null))
        break
      }
      case 2:{
        this.TORDepthTwo = this.$deepCopy(this.TORDepthOne) as VisualTreeNodePathModel[]
        this.TORDepthTwo.push(new VisualTreeNodePathModel(entityData.NodeId, entityData.EntityId,
        entityData.ParentNode.EntityId, null, entityData.Label));
        break
      }
      case 3:{
        this.TORDepthThree = this.$deepCopy(this.TORDepthTwo) as VisualTreeNodePathModel[]
        this.TORDepthThree.push(new VisualTreeNodePathModel(entityData.NodeId, entityData.EntityId,
            entityData.ParentNode.EntityId, null, entityData.Label));
        this.TORDepthThree[this.TORDepthThree.length-1].Name = entityData.EntityName
        break
      }
      case 4:{
        this.TORDepthFour = this.$deepCopy(this.TORDepthThree) as VisualTreeNodePathModel[]
        this.TORDepthFour.push(new VisualTreeNodePathModel(entityData.NodeId, entityData.EntityId,
            entityData.ParentNode.EntityId, null, entityData.Label));
        this.TORDepthFour[this.TORDepthFour.length-1].Name = entityData.EntityName
      }
    }
    if(this.TORDepthThree !=null){
      this.TORNodePathOnSection = this.$deepCopy(this.TORDepthThree) as VisualTreeNodePathModel[]
      this.emitNavigateDeep(this.TORNodePathOnSection);
      if(this.TORDepthFour !=null){
        this.TORNodePathOnString = this.$deepCopy(this.TORDepthFour) as VisualTreeNodePathModel[]
        console.log(this.TORNodePathOnString,"валидная модель 4 уровень");
        this.emitNavigateDeep(this.TORNodePathOnString);
        this.TORDepthFour = null;
      }
    }
  }



  //--------------- переменные

  // нужен для парса entityModel.Attributes
  private entityId: number = null;

  // нужна для парса атрибутов entityModel.Attributes
  private sortedAttributes: Array<Entity_AttributeValueModel | ConnectionReadModel> = [];

  // че то там тип сущности (полагаю это Id класса сметной строки по форме 2П в нашем случае)
  private entityTypeId: number | null = null;

  // модель для сохранения данных из entityMasterModel, передается в submitEntity()
  private entityModel: EntityModel = null;

  // версионируемая модель, нужна для отката к предыдущей модели я так понял
  private entityVersionModel: EntityModel = null;

  // модель хранения данных редактируемого/добавляемого экземпляра
  public entityMasterModel: EntityMasterModel = null;

  // модель для отката
  private entityMasterModelForRollback: EntityMasterModel = null;

  private entityTypesString : any = null;

  // текущее Id типа сущности
  private localEntityTypeId: number = null;

  private selectedNodePath: VisualTreeNodePathModel[] = [];

  private formOptions: GlobalDisplayFormParams = null;

  //текущий Id сущности
  private localEntityId: number = null;

  // текущий nodePath
  private localNodePath: VisualTreeNodePathModel[] = [];

  // entityEditorData для модели entityMasterModel перед открытием диалога MultiSelectTreeView
  public dataForEntityModel: EntityEditorData = null;

  // id группы связей
  public connectionGroupId: number;

  // связь
  public connection : ConnectionReadModel[];

  private entityTypeModel : VisualTreeNodePathModel[];

  private entityTypeModelDown : VisualTreeNodePathModel[];

  // перехват изменения nodePath
  @Watch("nodePath", { deep: true })
  async nodePathChanged() {
    this.selectedEntityData = null;
    this.selectedEntityName = "";

    let tableNode = await this.getExtendedNode(
        this.nodePath[this.nodePath.length - 1].NodeId,
        this.nodePath
    );

    this.tableNodeHidden = tableNode.Hidden;
    this.tableNodeCreateAllowed = tableNode.CreateAllowed;
    this.tableNodeEditAllowed = tableNode.EditAllowed;

    //await this.mountToolbarItems();
  }

  // nodePathChanged и getExtendedNode нужны для перерисовки тулбара
  async getExtendedNode(nodeId: number, nodePath: VisualTreeNodePathModel[]) {
    let nodeQueryModel: VisualTreeNodeQueryModel = new VisualTreeNodeQueryModel(
        {
          TreeId: this.treeId,
          Parameters: new VisualTreeNodeQueryParameter({
            Nodes: nodePath,
            Skip: 0,
            Top: 15
          })
        }
    );
    let extendedPathResponse = await this.visualTreeController.GetExtendedNodes(
        nodeQueryModel
    );
    return extendedPathResponse.data.Data.find(
        n => n.NodeId == nodeId
    );
  }

  async getNode(nodePath: VisualTreeNodePathModel[]) {
    let nodeQueryModel: VisualTreeNodeQueryModel = new VisualTreeNodeQueryModel(
        {
          TreeId: this.treeId,
          Parameters: new VisualTreeNodeQueryParameter({
            Nodes: nodePath,
            Skip: 0,
            Top: 15
          })
        }
    );
    let response = await this.visualTreeController.GetNodes(
        nodeQueryModel
    );
    return response.data.Data
  }


  emitNavigateDeep(nodePath: VisualTreeNodePathModel[]): void {
    this.$emit("navigateDeep", nodePath);
  }


  async factsAdd() {

    await this.showCard();
    this.dialog = !this.dialog;
  }


  async bussinesAdd() {
    this.dialog = !this.dialog;
    await this.showCardTotal();
  }

  // getEntityTypeID
  async showCard(){

   let possibleEntityTypes = await this.visualTreeController.GetEntityTypes(
        this.treeId,
        this.TORNodePathOnSection
    );

   let model =  this.$deepCopy(this.TORNodePathOnSection) as VisualTreeNodePathModel[]
   model.pop()
   let entityTypeNode  = possibleEntityTypes.data.Data.find(capt => capt.Caption == "ТОР. Строка фактических затрат")
    this.showEntityEditorForCreation(entityTypeNode, model);
  }

  showEntityEditorForCreation(entityType: EntityTypeNodeModel, nodePath: VisualTreeNodePathModel[]) {
    this.selectedNodePath = nodePath;

    this.formOptions = new GlobalDisplayFormParams({
      DisplayType: 15,
      Title: "test",
      Data: EntityEditorData.GetForCreation(entityType.EntityTypeId, entityType.NodeId, nodePath),
      NodePath: nodePath,
      TreeId: this.treeId,
      CreatingNodeId: entityType.NodeId,
    });
  }

  public getKey(key: string): string {
    return this.$t(key) as string;
  }

  getCreateDisplayType(displayType: DisplayTypeEnum) {
    if (!displayType) {
      return DisplayTypeEnum.EntityFormCreation;
    }
    if (displayType === DisplayTypeEnum.ComplexTableForm ||
        displayType === DisplayTypeEnum.EntityForm ||
        displayType === DisplayTypeEnum.OneCascadeForm ||
        displayType === DisplayTypeEnum.BaseEntityForm ||
        displayType === DisplayTypeEnum.Cascade3Form) {
      return DisplayTypeEnum.EntityFormCreation;
    }

    return displayType;
  }


  async showCardTotal(){

    let possibleEntityTypes = await this.visualTreeController.GetEntityTypes(
        this.treeId,
        this.TORNodePathOnSection
    );

    let model =  this.$deepCopy(this.TORNodePathOnSection) as VisualTreeNodePathModel[]
    model.pop()
    let entityTypeNode  = possibleEntityTypes.data.Data.find(capt => capt.Caption == "ТОР. Строка командировочных расходов")
    this.showEntityEditorForCreation(entityTypeNode, model);
  }



  //------------ otherButtons ------------

  //------------ MultiSelectTreeView ------------

  // триггер переключения модалки
  public treeDialogOpened: boolean = false;

  // выбранные связи в дереве в правоц части
  private preparedValuesFromTreeSelection: IdNameTypeModel[] = new Array<IdNameTypeModel>(0);

  // переменная для парса связей для их сохранения в entityModel
  private localValues: ConnectionValueEntity[] = new Array<ConnectionValueEntity>(0);

  private connectionIds : number[]

  // переменная, которая вероятно не испольщуется
  //private connectionValuesFromDialog: ConnectionValueModel[];

  // отслеживание изменения EntityEditorData для EntityMasterModel
  @Watch("treeDialogOpened", { immediate: true, deep: true })
  dialogStateChanged() {
    if (this.treeDialogOpened == false)
    {
      // зануляем значения выбранные справа при следующем открытии диалога
      this.preparedValuesFromTreeSelection = null;
    }
  }
  // событие добавления расценки через MultiSelectTreeView (tree)
  async estimatedStringAdd() {
    this.treeDialogOpened = await this.preparedDataForDialog();
  }

  // ждем выбора entityTypeNode
  async preparedDataForDialog()
  {
    // TODO Пока что багает, чтобы работало, сначала надо нажать на любую
    // TODO из трех других кнопок и тогда работает правильно.
    // TODO Сделал эту штуку, чтобы игнорировать и не вызывать entityTypePicker
    // TODO надо разобраться как navigate работает у дефолтного события добавления
    // получаем массив возможных классов(сущностей) для создания

    this.possibleEntityTypes = await this.visualTreeController.GetEntityTypes(
        this.treeId,
        this.TORNodePathOnSection
    );

    let sectionName = "ТОР. Строка на основе справ. РиУ, КО и ОИ"


    // находим нужный id по Caption(Название класса)
    this.entityTypeNode = this.possibleEntityTypes.data.Data.find(capt => capt.Caption == sectionName)
    // получаем массив возможных nodeId классов(сущностей) для создания
    // let nodes = await this.visualTreeController.GetMasterModel(
    //     entityTypeNode.EntityTypeId,
    //     this.treeId,
    //     this.nodePath
    // );
    // console.log("nodes", nodes)
    // передаем EntityEditorData для диалога (entityMasterModel)
    this.dataForEntityModel = new EntityEditorData(this.entityTypeNode.EntityTypeId, null, this.TORNodePathOnSection)
    console.log("dataForEntityModel", this.dataForEntityModel)

    // не уверен этот ли нужно передавать
    this.entityTypeId = this.entityTypeNode.EntityTypeId

    return true;
  }


  // изменение выбранных значений
  multiselectValuesChange(values: IdNameTypeModel[]): void {
    this.preparedValuesFromTreeSelection = values;
  }

  // передать выбранные значения в правую часть
  private getSelectedValuesForTree(): IdNameTypeModel[] {
    return this.localValues.filter(x => x.EntityId > 0).map(x => new IdNameTypeModel(x.EntityId, x.EntityName, x.EntityTypeId));
  }

  // сохранить значения
  applySelectedValuesFromTree(): void {
    let preparedConnectionValues = new Array<ConnectionValueEntity>();
    if (this.preparedValuesFromTreeSelection.length == 0) {
      preparedConnectionValues.push(new ConnectionValueEntity(null, null, null));
    }
    else {
      this.preparedValuesFromTreeSelection.forEach(x => {
        preparedConnectionValues.push(new ConnectionValueEntity(x.Id, x.Name, x.TypeId));
      });
    }

    // парсим связь
    let entityIds: number[] = [];
    preparedConnectionValues.forEach(t => {entityIds.push(t.EntityId)})

    this.localValues = preparedConnectionValues;

    console.log("valuesFromMultiSelectTreeView",this.localValues)

    if (this.entityModel.Connections) {
      this.entityModel.Connections.push(new ConnectionValueModel(this.connectionGroupId, entityIds));
    }

    //тут по идее надо модель заносить в дерево и сохранять
    this.submitEntity()

    // закрываем диалог дерева
    this.treeDialogOpened = false;
  }

  //------------ MultiSelectTreeView ------------

  //-------------- Подготовка данных для диалога (entityMasterModel)

  // отслеживание изменения EntityEditorData для EntityMasterModel
  @Watch("dataForEntityModel", { immediate: true, deep: true })
  dataForEntityModelChanged() {

    if (this.dataForEntityModel) {
      // заносим инфу
      this.localEntityId = this.dataForEntityModel.EntityId;
      this.localEntityTypeId = this.dataForEntityModel.EntityTypeId;
      this.localNodePath = this.dataForEntityModel.NodePath;
      // вызываем метод создания EntityMasterModel
      this.getMasterEntityModel();
    }
  }


  async getStringType(){

    let nodeModelDown = this.$deepCopy(this.nodePath) as VisualTreeNodePathModel[]
    nodeModelDown.pop()

    this.entityTypesString = await this.visualTreeController.GetEntityTypes(
        this.treeId,
        nodeModelDown
    );


    let nodeType = this.entityTypesString.data.Data.find(capt => capt.Caption == "Карточка ТОР (экземпляр)")
    let dataForEntityModel = new EntityEditorData(nodeType.EntityTypeId, null, nodeModelDown)

   let model = await this.entityController.ProtectedGetMasterModel(dataForEntityModel.EntityTypeId, nodeModelDown[3].EntityId, this.treeId, nodeModelDown)
    console.log(model)
    for (let modelElement of model.data.Data[0].AttributeValues) {
      switch (modelElement.ValueStr) {
          case "Строка на основе РиУ, КО и ОИ" :
            return 1
          case "Строка фактических затрат" :
            return 2
          case "Командировочные расходы" :
            return 3
      }
    }
  }

  // получить EntityMasterModel
  getMasterEntityModel() {
    this.entityModel = null;
    this.entityVersionModel = null;

    if (this.localEntityId > 0) {
      if (this.treeId && this.localNodePath) {
        this.entityController.ProtectedGetMasterModel(this.localEntityTypeId, this.localEntityId, this.treeId, this.TORNodePathOnSection)
            .then((response) => {this.gotMasterEntityModel(response);})
            .catch(error => {this.showError(error);
            });
      } else {
        this.visualTreeController.GetMasterModelForNodeId(this.localEntityTypeId, this.treeId, this.TORNodePathOnSection, this.entityTypeNode.NodeId)
            .then(this.gotMasterEntityModel)
      }
    }
    else {
      console.log("localNodePath",this.localNodePath)
      this.visualTreeController.GetMasterModelForNodeId(this.localEntityTypeId, this.treeId, this.TORNodePathOnSection, this.entityTypeNode.NodeId)
          .then((response:any) => { this.gotMasterEntityModel(response);})
          .catch(error => { this.showError(error) });
    }
  }

  // обработать полученную модель
  gotMasterEntityModel(response: AxiosResponse<PageModel<EntityMasterModel>>) {
    console.log("gotMasterEntityModel_response in PirMain",response);
    this.entityMasterModel = new EntityMasterModel();

    let preparedMasterModel = response.data.Data[0] as EntityMasterModel;

    this.entityNameUpdated(preparedMasterModel.EntityName);

    if (!(preparedMasterModel.EntityId > 0)) {
      //EntityEditorHelper.addDefaultValuesToModel(preparedMasterModel, this.dataForEntityModel.DefaultValues, this.DefaultConnectionValues);
    }

    this.entityMasterModel = preparedMasterModel;
    console.log("entityMasterModel",this.entityMasterModel)
    this.entityMasterModelForRollback = Clonning.deepCopy(preparedMasterModel);

    this.entityInfoChanged(new EntityEditorInfoObject(
        preparedMasterModel.EntityType.Id,
        preparedMasterModel.EntityType.Caption,
        this.dataForEntityModel.EntityId,
        preparedMasterModel.EntityName
    ));


    //this.entityMasterModel.Connections.forEach(t => {this.connectionIds.push(t.Id);})


    this.connectionGroupId = this.entityMasterModel.Connections.find(conn => conn.Caption == "Строка справочника РиУ, КО и ОИ").Id;



    this.connection = this.entityMasterModel.Connections


  }

  // отслеживаем изменение entityMasterModel
  @Watch("entityMasterModel", { deep: true, immediate: true })
  masterModelChanged() {

    if (this.entityMasterModel == null) {
      return;
    }
    let attributesTemp = Clonning.deepCopy(this.entityMasterModel.AttributeDefinitions) as Array<Entity_AttributeValueModel>;

    this.createAttributesHierarchy(attributesTemp);
    this.sortAttributes(attributesTemp);

    // создаем и заполняем entityModel для сохранения
    this.entityModel = this.createModel();
  }

  // определеям иерархию для атрибутов
  createAttributesHierarchy(attributesTemp: Array<Entity_AttributeValueModel>): Array<Entity_AttributeValueModel> {
    let attributes = attributesTemp;

    attributes.forEach(x => {
      if (x.Value == undefined) {
        x.Value = null;
      }
    });

    let tmpEntityId = this.entityId;
    attributes.forEach(attribute => {
      if (attribute.DefaultValue && !attribute.Value && !(tmpEntityId > 0)) {
        attribute.Value = attribute.DefaultValue;
      }
    });
    return attributes;
  }

  // сортируем атрибуты для парса
  sortAttributes(definitions: Entity_AttributeValueModel[])
  {
    let preparedAttributes = [];
    if(definitions && definitions.length){
      definitions.forEach(def => {
        def.IsAttribute = true;
        preparedAttributes.push(def)
      });
    }

    if (this.entityMasterModel.Connections && this.entityMasterModel.Connections.length) {
      let clonningConnections = Clonning.deepCopy(this.entityMasterModel.Connections) as ConnectionReadModel[];
      clonningConnections.forEach(con => {con.IsAttribute = false; preparedAttributes.push(con)});
    }

    preparedAttributes = AttributeEditorHelper.SortItemsByAscendingPosition(preparedAttributes);

    AttributeEditorHelper.specifyAttributeIndexInsideAttributeSection(preparedAttributes);

    // получаем атрибуты для передачи в entityModel
    this.sortedAttributes = preparedAttributes;
  }

  async giveSectionName(sections: string[], key: number, nodePath : VisualTreeNodePathModel[]){

    this.possibleEntityTypes = await this.visualTreeController.GetEntityTypes(
        this.treeId,
        nodePath
    );

    for (const section of sections) {
      if(this.possibleEntityTypes.data.Data[key].Caption == section){
        console.log(section)
        return section
      }
    }
  }


  // создать entityModel
  createModel(): EntityModel {
    let model = new EntityModel();
    model.Attributes = [];
    model.TypeId = this.entityMasterModel.EntityType.Id;

    this.sortedAttributes.filter(x => x.IsAttribute).forEach(attr => {
      model.Attributes.push(new AttributeEditorModel((<Entity_AttributeValueModel>attr).Value as string, attr.Id));
    });


    model.Connections = this.getConnections();


    return model;
  }

  // Распарсить и получитья связи для entityModel
  getConnections()
  {
    let model2 = new Array<EntityConnectionValue>();
    let model3 = new Array<ConnectionValueModel>();

    this.entityMasterModel.ConnectionValues.forEach(t => {model2.push(t);})

    let entityIds: number[] = [];
    for (let itm of model2)
    {
      itm.Entities.forEach(t =>{
        entityIds.push(t.EntityId)
      })
      model3.push(new ConnectionValueModel(itm.ConnectionGroupId, entityIds))
    }

    return model3;
  }

  // Сохранить данные в entityModel
  submitEntity(): Promise<any> {


    // создаем модель через entityController
    if (this.entityModel) {
      this.preloaderState = !this.preloaderState
      //return this.entityController.createEntity(this.entityModel)
      return this.createEntity(this.entityModel) //visualTreeNodeIdForCreation = 41628
          .then(this.entityCreatedForModel)
          .catch(error => {
            this.showError(error);
          });
    } else {
      console.log("emptyEntityModel, unable to save")
    }
  }

  //
  createEntityProtected(entity: EntityModel, visualTreeNodeIdForCreation: number): AxiosPromise<IdNameWarningModel> {
    return Axios.post(SiteUrl.Entity_Create(visualTreeNodeIdForCreation), entity);
  }
  // создаем сущность
  private async createEntity(entity: EntityModel): Promise<IIdNameNodePathModel> {
    let response = await this.createEntityProtected(entity, this.entityTypeNode.NodeId);

    let createdEntityNodePath: VisualTreeNodePathModel[] = [];
    createdEntityNodePath.push(...this.localNodePath);
    createdEntityNodePath.push({
      EntityId: response.data.Id,
      Label: null,
      Name: response.data.Name,
      NodeId: this.entityTypeNode.NodeId,
      Payload: null,
      ParentNodeId: this.localNodePath.length > 0 ? this.localNodePath[this.localNodePath.length - 1].NodeId : null
    });

    return {
      Id: response.data.Id,
      Name: response.data.Name,
      NodePath: createdEntityNodePath,
      Warnings: response.data.Warnings
    };
  }
  // сохраняем модель и уведомляем пользователя о сохранении
  async entityCreatedForModel(response: IIdNameNodePathModel): Promise<EntityCreatedEventArg> {
    this.preloaderState = !this.preloaderState
    await (this.$refs.mainTable as any).refresh();
    console.log("entityCreatedForModel_response", response)
    //this.entityMasterModelForRollback = (await this.entityController.ProtectedGetMasterModel(this.localEntityTypeId, response.Id, this.treeId, response.NodePath)).data.Data[0];
    //console.log("entityMasterModelForRollback", this.entityMasterModelForRollback)

    this.entityNameUpdated(response.Name);
    this.showSuccess(this.$t('Расценка успешно добавлена') as string);

    // TODO разобраться как рефрешить таблицу
    //this.refresh();

    return {
      Id: response.Id,
      Name: response.Name,
      NodePath: this.localNodePath
    };
  }

  // update вообще не уверен, что нужен
  async entityUpdated(response: AxiosResponse<IdNameWarningModel>): Promise<IdNameWarningModel> {
    //this.loading = false;

    this.entityMasterModelForRollback = (await this.entityController.ProtectedGetMasterModel(this.localEntityTypeId,
        this.localEntityId, this.treeId, this.localNodePath)).data.Data[0];
    //this.newEntityCreated();

    this.entityNameUpdated(response.data.Name)

    return response.data;
  }

  // метод, вероятно не пригодится
  entityNameUpdated(name: string): string {
    return name;
  }

  // метод, вероятно не пригодится
  entityInfoChanged(data: EntityEditorInfoObject) {
    return data;
  }

  // уведомление для ошибки
  showError(message) {
    this.preloaderState = !this.preloaderState
    if (this.$notification) {
      this.$notification.NetworkError(message);
    }
    else {
      this.$emit("notification", message);
    }
  }

  // уведомление в случае успешной операции
  showSuccess(message) {
    if (this.$notification) {
      this.$notification.Success(message);
    }
  }

}
</script>

<style>
</style>
